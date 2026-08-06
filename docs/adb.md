# ADB device sharing

Turn on *Tunnels → Share ADB devices* in the tray and every Android device your
Windows machine can see — USB or wireless — becomes visible to `adb` on
the server. Useful when Claude Code runs remotely but the phone is on
your desk.

## How it works

There is no per-device forwarding. The tray opens one reverse tunnel
that exposes your **local adb server socket** on the server:

```
ssh -R 5037:localhost:5037 myserver
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

## Limitations

- Reconnect retries the last known `ip:port` only. If the port
  randomized (Android 11+ wireless debugging after a toggle), reconnect
  from the phone once — or use `adb tcpip 5555` and never think about it.
- `adb` must be on PATH on the Windows side.
