# SSH Load Balancing via nginx (stream module)

Single entry point for SSH: users connect to **one IP** (`172.21.129.16`), and
nginx transparently load-balances the TCP connection across two backend SSH
daemons, capping the primary server at 5 concurrent connections and spilling
the rest to the secondary server.

```
                 ssh root@172.21.129.16
                          |
                          v
        +---------------------------------------+
        |   172.21.129.16 : nginx (stream mod)   |
        |   listens on :22, upstream "ssh_pool"  |
        +---------------------------------------+
             |  up to 5 conns          overflow (backup)
             v                              v
   127.0.0.1:2222 (local sshd)     172.21.129.138:22 (sshd)
   same box as nginx               second RHEL host
```

## Why this design

- **nginx stream module** (`ngx_stream_module`, TCP/L4 proxy — not `http{}`)
  proxies raw TCP, so it works for any protocol, including SSH.
- **Local sshd moved to `127.0.0.1:2222`** because nginx needs port 22 on
  172.21.129.16 for itself; sshd is bound to loopback only, so it is not
  directly reachable from the network — all traffic must go through nginx.
- **`max_conns=5` + a shared memory `zone`** in the upstream block caps the
  primary backend at 5 *simultaneous* connections. The `zone` directive is
  required — without it, `max_conns` is tracked per nginx worker process, so
  "5" would really mean `5 x worker_processes` before overflow.
- **`backup` on the second server**, not plain round-robin. Round-robin would
  split traffic roughly 50/50 from the very first connection. Marking
  172.21.129.138 as a `backup` server makes nginx fill the primary to its
  `max_conns` limit first, and only then spill new connections to the backup
  — i.e. "fill-then-spill", which is what "max 5 on the main server, rest go
  to server 2" actually means.
- **Both backends share identical SSH host keys** (rsa/ecdsa/ed25519, copied
  from server 1 to server 2). This is required for a Layer-4 SSH load
  balancer: nginx proxies raw bytes and never terminates SSH, so each backend
  exposes its own host key straight to the client. With different keys, a
  client that connects twice and gets routed to different backends sees the
  host identity "change" under the same IP — which triggers OpenSSH's
  man-in-the-middle warning and refuses the connection. Sharing host keys
  makes the pair present one consistent identity, so clients only ever
  approve/pin the key once, regardless of which real backend they land on.
  (Private host keys were copied directly between the two root shells over
  SSH on the private lab network; they are intentionally **not** included in
  this repository.)
- **SELinux stays Enforcing.** nginx runs under the `httpd_t` domain, which is
  not allowed by default to bind/connect to `ssh_port_t` (port 22/2222). A
  minimal custom policy module (`selinux/ssh_lb_nginx.te`) grants exactly one
  rule — `allow httpd_t ssh_port_t:tcp_socket name_bind;` — generated via
  `audit2allow` from the real AVC denial, instead of disabling enforcement.

## File locations on the actual RHEL hosts

| File in this repo | Path on the real host | Host |
|---|---|---|
| `server1-172.21.129.16/nginx.conf` | `/etc/nginx/nginx.conf` | 172.21.129.16 (nginx / LB) |
| `server1-172.21.129.16/ssh-lb.conf` | `/etc/nginx/stream.conf.d/ssh-lb.conf` | 172.21.129.16 |
| `server1-172.21.129.16/sshd_config` | `/etc/ssh/sshd_config` | 172.21.129.16 (local backend, port 2222/loopback) |
| `server1-172.21.129.16/firewalld-public-zone.txt` | output of `firewall-cmd --list-all` | 172.21.129.16 |
| `server2-172.21.129.138/sshd_config` | `/etc/ssh/sshd_config` | 172.21.129.138 (overflow backend, standard port 22) |
| `server2-172.21.129.138/firewalld-public-zone.txt` | output of `firewall-cmd --list-all` | 172.21.129.138 |
| `selinux/ssh_lb_nginx.te` / `.pp` | installed via `semodule -i` | 172.21.129.16 |

Private SSH host keys, `/etc/shadow`, and account passwords are **not**
included in this repository.

## Test performed

Test users `lbtest1`..`lbtest5` (password-only, no login shell home
directory) were created identically on both backends. 8 concurrent SSH
sessions were then opened against the single load-balancer IP
(`172.21.129.16:22`) and held open for ~20s each, mixing `root` and the 5
test accounts.

Result, read from nginx's `stream_access.log` (each line = one proxied
connection and which backend it went to):

```
upstream_addr=172.21.129.138:22   (overflow)
upstream_addr=172.21.129.138:22   (overflow)
upstream_addr=127.0.0.1:2222      (primary)
upstream_addr=127.0.0.1:2222      (primary)
upstream_addr=127.0.0.1:2222      (primary)
upstream_addr=127.0.0.1:2222      (primary)
upstream_addr=127.0.0.1:2222      (primary)
upstream_addr=172.21.129.138:22   (overflow)
```

**5 connections landed on the primary (172.21.129.16, local :2222) and the
remaining 3 overflowed to the secondary (172.21.129.138:22)** — confirmed
independently by counting live established sockets on both backends mid-test
(`ss -tn state established`). This matches the required behavior: max 5
connections on the main server, the rest routed to server 2.

## Known limitations (lab setup, not production-hardened)

- Root login over SSH with a shared password is enabled for this exercise.
  For anything beyond a lab, use key-based auth and disable
  `PermitRootLogin`/`PasswordAuthentication`.
- No active health checks (open-source nginx only supports passive checks via
  `max_fails`/`fail_timeout`, both configured here).
- Both backends must be kept in sync (users, host keys) manually; a real
  deployment would automate this (e.g. central auth, config management).
