[tether](../README.md) · [Tunnels](tunnels.md) · Phones over adb · [Android](android.md)

# Phones over adb

When Claude Code runs on a server and the phone is on your desk, tether lets `adb` on the server use
the phone: `adb devices`, `adb install`, `adb shell` and logcat all work as if it were plugged in there.

## Turn it on

1. Connect the phone to the PC by USB, or with `adb connect` over Wi-Fi. `adb` must be on the PATH on
   Windows.
2. Right-click the tray icon and choose **Tunnels → Share phones (adb)**. The menu shows how many phones
   the server can see.
3. On the server, `adb devices` lists the phone.

**Settings → Phones (adb)** has the same switch, the **Server** to share with, and the **adb port on
the server** (5037 unless you change it).

While sharing is on, any program or user on the server can use the shared phones with adb: the adb port
opens on the server's `127.0.0.1`, with no login.

## Open a server port from the phone

To open a dev site on the server (say port 3000) in the phone's browser, put `3000` in **Phone ports**
on the Phones (adb) page. Several ports are fine: `3000, 6111`. While sharing is on, each phone's
`localhost:3000` reaches port 3000 on the server. The tray shows it under **Share phones (adb)** as
"Phone's localhost:3000 → server", with the reason under it when it is down.

## Set up the server

If the server runs its own adb, port 5037 is already in use there and sharing cannot start. Either:

- set **adb port on the server** to `5038`, and on the server
  `export ANDROID_ADB_SERVER_PORT=5038` before running `adb`; or
- stop the server's own adb with `adb kill-server`, before you turn sharing on.

> [!WARNING]
> Never run `adb kill-server` on the server while sharing is on. The server's `adb` is then talking to
> the adb on your PC, and that is the one it would stop.

> [!IMPORTANT]
> The server's `adb` and the PC's must be the same version (`adb --version` on both). On a mismatch, the
> server's `adb` restarts the adb it talks to, which is the one on your PC, and phones disappear from
> both. Install the same platform-tools version on both machines.

## Keep wireless debugging connected

tether reconnects a phone that drops off Wi-Fi, at the address it last had. Android 11's
**Wireless debugging** picks a new port each time it is turned on, so after a drop that address may be
gone. A fixed port avoids it. With the phone connected once (by USB or Wi-Fi):

```
adb tcpip 5555
adb connect <phone-ip>:5555
```

Port 5555 stays until the phone restarts. Most other drops come from one of these, on the phone or the
router:

- battery optimisation for wireless debugging: turn it off, or keep the screen on
- the phone's IP address changing: give it a fixed one (a DHCP reservation)
- AP or client isolation on the router: turn it off
- the phone and the PC on different bands or networks

## If `adb devices` on the server shows nothing, or hangs

tether fixes this by itself within a few seconds: it finds what holds port 5037 on the server and
clears it. To do it by hand, run `ss -tlnpe | grep 5037` on the server:

- **`sshd`**, from a session that is not your current one: it is an old connection's leftover. End it
  with `loginctl kill-session <N>`, or `kill` its process.
- **`adb`**: the server's own adb started while sharing was down. Stop it with
  `timeout 5 adb kill-server`.

Leftover connections become rare once the server's sshd notices dead clients: set
`ClientAliveInterval 15` and `ClientAliveCountMax 3` in its `sshd_config`
([Tunnels](tunnels.md#when-a-tunnel-wont-stay-up)).

## How it works

There is no forwarding per phone. tether opens one tunnel that makes the adb on your PC answer on the
server's port 5037:

```
ssh -R 5037:127.0.0.1:5037 myserver
```

The server's `adb` connects to it as if it were a local adb. tether restarts this tunnel the same way as
any other ([Tunnels](tunnels.md#how-it-works)). While sharing is on, tether also runs `adb devices` every
5 seconds and reconnects any phone on Wi-Fi that dropped.

Phone ports take two hops: `adb reverse` on each phone (the phone's port to the PC's), and one more
tunnel from the PC to the server. tether runs `adb reverse` again when a phone appears, after the ports
change, and once a minute, because adb forgets it each time a phone reconnects. If a tunnel you already
have opens the same port to the same server, tether leaves the port to it.
