# Observing and Verifying SSH Load-Balanced Sessions (this repo's architecture)

This repo's architecture has **no VIP and no HA** — a single nginx instance
runs on `172.21.129.16` and is the only entry point (that's the known SPOF
this repo documents; see [nginx_SPF](https://github.com/dev1237/nginx_SPF)
for the fix). Users connect with `ssh user@172.21.129.16` directly. The
backend addresses are specific to this repo too: the local backend is
`127.0.0.1:2222` (sshd moved off :22 so nginx could bind it), the overflow
backend is `172.21.129.138:22`.

The general methodology below is identical to the fuller writeup in
[nginx_round-robin's OBSERVING-LOAD-BALANCED-USERS.md](https://github.com/dev1237/nginx_round-robin/blob/main/OBSERVING-LOAD-BALANCED-USERS.md)
(itself a fully worked 10-user example with real captured output for every
method) — read that one for the complete walkthrough, including two
important caveats that apply here unchanged:

1. `who`/`w`/`last` only see a session if the SSH connection got a
   pseudo-terminal — force one with `ssh -t -t` if testing non-interactively.
2. The backend never sees the true client IP (nginx re-originates the
   connection); only nginx's own `stream_access.log` logs the real client.

## Commands specific to this repo's addresses

```bash
# Live connection count on the primary (local) backend -- note port 2222, not 22
ss -tn state established '( sport = :2222 )' | tail -n +2 | wc -l

# Live connection count on the overflow backend, over SSH
ssh root@172.21.129.138 "ss -tn state established '( sport = :22 )' | tail -n +2 | wc -l"

# Per-user live view (needs -t -t on the client side to populate)
who
w
```

## Real captured result (from this repo's own test, see its README)

8 concurrent SSH connections to `172.21.129.16` (root + 5 `lbtest*` users),
read from `/var/log/nginx/stream_access.log`:

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

5 landed on the primary (matches `max_conns=5`), 3 overflowed — confirmed
independently at the time via `ss` connection counts on both backends
(see this repo's main README for the full test writeup).

**Note on scope:** that original test verified counts via `ss` and the
nginx log. It did not specifically capture a per-username `who`/`w`
snapshot or sshd auth-log excerpt for *this* repo's exact addresses — rather
than fabricate example output for those two methods here, use the exact
same commands shown in `nginx_round-robin`'s guide (they're
architecture-independent — `who`, `w`, and `grep Accepted /var/log/secure`
work identically regardless of which port/IP the backend sshd is on) against
this repo's own primary (`ssh root@172.21.129.16` on box 1, since the
primary IS `172.21.129.16` here — no VIP redirect needed) and overflow
(`172.21.129.138`) hosts.
