# Tunnels

Each enabled tunnel is one supervised `ssh -N` process:

```
ssh -N -o BatchMode=yes -o ConnectTimeout=10 -o ExitOnForwardFailure=yes \
    -o ServerAliveInterval=15 -o ServerAliveCountMax=3 \
    -L 3000:localhost:3000 myserver
```

- `ExitOnForwardFailure` — if the forward cannot bind, ssh exits instead
  of sitting there half-working; the supervisor retries with backoff.
- `ServerAliveInterval 15` / `CountMax 3` — a dead network kills the
  process within ~45 s, which triggers a restart.
- Backoff: 1 s doubling to a 60 s cap. A tunnel that held for 30+ seconds
  resets the backoff, so a one-off drop reconnects in ~1 s.
- Sleep/resume: on Windows resume the supervisor kills every tunnel
  process immediately (their sockets are dead anyway) so they restart in
  about a second instead of waiting out the keep-alive timeout.

## Config format

`%APPDATA%\tether\config.ini`, one line per tunnel:

```
tunnel=web|myserver|L|3000, 5173|on
```

Fields: name, ssh-config host alias, `L` or `R`, ports, `on`/`off`.
A bare port `3000` expands to `3000:127.0.0.1:3000` — explicitly IPv4,
because Windows resolves `localhost` to `::1` and IPv4-only targets
(adb, many dev servers) then fail inside the tunnel. Anything containing
`:` is passed to ssh unchanged, so `8080:10.0.0.5:80` or
`0.0.0.0:3000:localhost:3000` work as usual.

## Edge cases

**Remote (`R`) port stays "in use" after a drop.** When a reverse tunnel
dies, the remote sshd can keep the port bound until it notices the dead
connection, so the reconnect fails with *address already in use*. The
supervisor just keeps retrying — the port frees once sshd's own timers
fire. To make that fast, set on the server:

```
# /etc/ssh/sshd_config
ClientAliveInterval 15
ClientAliveCountMax 3
```

**Local port already taken.** `ExitOnForwardFailure` makes ssh exit and
the tunnel shows ↻ with the error in its tooltip. Free the port or pick
another; the supervisor picks the tunnel up on its next retry.

**Password prompts.** Tunnels run with `BatchMode=yes`: key-based auth
only, same as screenshot uploads. A host that asks for a password will
show ↻ forever — check the tooltip.

**Wi-Fi switch / VPN toggle.** Looks like a drop: keep-alive kills the
process within ~45 s and the backoff loop reconnects. Nothing to do.

**A reverse (`R`) tunnel shows ● but clients can't connect.** The live
dot for reverse tunnels means the ssh process is up — it cannot see the
remote port. If something on the server grabbed the IPv4 side of the
port before the tunnel bound, sshd may hold only `::1` and IPv4 clients
hit the squatter. Check `ss -tlnp | grep <port>` on the server; free
the port and the supervisor rebinds on its next retry. (The ADB share
handles its own port this way automatically.)
