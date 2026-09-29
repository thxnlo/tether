# ADB device sharing

Turn on *Tunnels → Share ADB devices* in the tray and every Android device your
Windows machine can see — USB or wireless — becomes visible to `adb` on
the server. Useful when Claude Code runs remotely but the phone is on
your desk.

## How it works

There is no per-device forwarding. The tray opens one reverse tunnel
that exposes your **local adb server socket** on the server:

```
ssh -R 5037:127.0.0.1:5037 myserver
```

`adb` on the server then talks to your local adb server, so `adb devices`,
`adb shell`, `adb install`, logcat — everything — works against your
local phones. The tunnel is supervised exactly like the port-forwarding
tunnels (keep-alive, backoff restart, resume kick — see
[tunnels.md](tunnels.md)).

While sharing is on, the tray also polls `adb devices` every 5 seconds
and auto-retries `adb connect ip:port` for any wireless device that
drops — phones lose wireless debugging constantly (sleep, Wi-Fi switch,
battery saver). The ADB menu shows the live device count and how many
are reconnecting.

## Server-side setup

If the server runs its **own** adb daemon, port 5037 is taken and the
tunnel cannot bind. Two options:

- stop it: `adb kill-server` on the server, or
- set *Remote port* to e.g. `5038` in Settings and tell the server's adb
  client to use it: `export ANDROID_ADB_SERVER_PORT=5038`

**Version mismatch is the sharp edge.** The adb client checks the server
version and, on mismatch, kills the server and starts its own — through
the tunnel that kills *your local* adb server and the remote one takes
over port 5037 locally-on-the-server instead. Symptom: devices vanish on
both ends. Fix: install the same platform-tools version on both machines
(`adb --version` must match).

## Making wireless debugging survivable

Android 11+ "Wireless debugging" picks a new random port every time it
toggles, and pairing info does not include the connect port — so after a
drop the old `ip:port` may be gone for good. The reliable path is legacy
TCP mode on a fixed port:

```
adb tcpip 5555        # once, with the phone on USB (or already connected)
adb connect <phone-ip>:5555
```

`5555` stays until the phone reboots, and the tray's reconnect loop can
always win the device back at the same address. Also worth doing on the
phone/router, since most "random" drops are one of these:

- disable battery optimization for wireless debugging / keep screen on
- give the phone a DHCP reservation so its IP never changes
- turn off AP/client isolation on the router
- keep phone and PC on the same band/subnet

## If `adb devices` on the server suddenly shows nothing, or hangs

Two things can own the remote port after a drop, and tether tells them
apart before touching anything (v3.0.4):

1. **sshd itself, holding the previous tunnel's forward.** A server
   without `ClientAliveInterval` never notices that the old client is
   gone and keeps the port bound for hours. Every `adb` command on the
   server then *hangs* — it dials a forward that leads nowhere — and so
   did tether's own recovery up to v3.0.3 (`adb kill-server` on the host
   went down the same dead forward and left a zombie per retry). Now the
   bind-failure fix reads the listener's session scope (`ss -e`), kills
   that stale sshd session of yours (its `cgroup.procs` is readable
   without root; `/proc/<pid>/fd` of a privsep child is not), and rebinds
   on the next retry — a few seconds, not hours.
2. **A server-local adb daemon**, spawned by the first remote `adb`
   command during the gap. Evicted with `timeout 5 adb kill-server`,
   bounded so it can never hang the loop again.

To force it by hand: `ss -tlnpe | grep 5037` on the server — `sshd` with
a `session-N.scope` that is not your live one is the stale forward
(`kill` its user-side pid, or `loginctl kill-session N`); `adb` is the
daemon (`adb kill-server`). Never run `adb kill-server` *on the server*
while the tunnel is healthy: the adb client is talking to your Windows
adb server through it, and it is that one you would kill.

Make the stale case rare by letting sshd reap dead clients (see
[tunnels.md](tunnels.md)): `ClientAliveInterval 15`,
`ClientAliveCountMax 3` in `sshd_config`.

## Limitations

- Reconnect retries the last known `ip:port` only. If the port
  randomized (Android 11+ wireless debugging after a toggle), reconnect
  from the phone once — or use `adb tcpip 5555` and never think about it.
- `adb` must be on PATH on the Windows side.
