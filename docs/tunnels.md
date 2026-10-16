[tether](../README.md) · Tunnels · [Phones over adb](adb.md) · [Android](android.md)

# Tunnels

tether keeps SSH tunnels (port forwards) open: it restarts one when the network drops, when Wi-Fi or a
VPN changes, and as soon as the PC wakes from sleep.

## Add a tunnel

1. Right-click the tray icon and choose **Tunnels → Add tunnel…**.
2. Type the port, say `3000`. Or paste a whole command, such as
   `ssh -L 3000:127.0.0.1:3000 myserver`, and tether fills in the rest.
3. Pick the direction and the server, then **Add tunnel**. It starts at once.

The directions:

| In tether | What it does | In ssh |
|---|---|---|
| This PC → server | Opens the port on this PC; it reaches the same port on the server. `http://localhost:3000` here opens the server's port 3000. | `-L` |
| Server → this PC | Opens the port on the server; it reaches the same port on this PC. | `-R` |
| SOCKS proxy on this PC | A proxy on this PC whose traffic leaves from the server. | `-D` |

**The server you send to** is the default server: the tunnel follows whatever **Send to server** is set
to. A tunnel to `localhost` or to this PC's own name is refused, since it would never reach the server.

## See how a tunnel is doing

**Settings → Tunnels** lists every tunnel with a switch and its state in words. The tray's **Tunnels**
menu shows the same, with a tick for each tunnel that is on:

| State | Means |
|---|---|
| Running | The tunnel is up and its port answers |
| Connected | A Server → this PC tunnel is up. tether cannot check the port on the server's side |
| Starting, Connecting… | ssh is starting |
| Retrying | The tunnel went down. The reason shows under it, such as "Port 3000 is already in use on this PC" |
| Not started | tether will not start it, and says why |
| Off | Switched off |

**Tunnels → Copy URL** copies `http://localhost:<port>` for a running tunnel; right-clicking the tunnel in
the menu does the same. **Import from ~/.ssh/config**, on the Tunnels page, adds the `LocalForward`,
`RemoteForward` and `DynamicForward` lines you already have, switched off.

## When a tunnel won't stay up

**"Port … is already in use on this PC."** Another program has the port. Close it or pick another
port; tether tries again by itself.

**A Server → this PC tunnel keeps retrying with "address already in use".** After a drop, the server's
sshd can hold the old port until it notices the client is gone, and without keep-alive settings that
takes hours. tether keeps retrying. To make it take seconds, add this to `/etc/ssh/sshd_config` on the
server and reload sshd:

```
ClientAliveInterval 15
ClientAliveCountMax 3
```

**"… refused this PC's SSH key."** Tunnels log in the same way as uploads, with a key and no password
prompt. Run `ssh <server> true` in a terminal: it must print nothing and ask for nothing. See
[Set up SSH for tether](../README.md#set-up-ssh-for-tether).

**A Server → this PC tunnel shows Connected, but nothing connects on the server.** Something on the
server may have taken the port first. Run `ss -tlnp | grep <port>` there, stop what holds it, and tether
binds the port on its next try.

## How it works

Each tunnel that is on is one `ssh` process, watched and restarted by tether:

```
ssh -N -o BatchMode=yes -o ConnectTimeout=10 -o ExitOnForwardFailure=yes
    -o ServerAliveInterval=15 -o ServerAliveCountMax=3
    -L localhost:3000:127.0.0.1:3000 myserver
```

- `ExitOnForwardFailure`: if the port cannot be opened, ssh exits instead of half-working, and tether
  retries.
- `ServerAliveInterval 15` with `CountMax 3`: a dead network ends the process within about 45 s,
  which starts a restart.
- Retries wait 1 s, then twice as long each time, up to 60 s. A tunnel that stayed up for 30 s starts
  again from 1 s, so a one-off drop reconnects in about a second.
- When Windows wakes from sleep, tether restarts every tunnel at once instead of waiting for the
  keep-alive to notice.

A bare port `3000` becomes `3000:127.0.0.1:3000`. tether writes `127.0.0.1`, not `localhost`, because
Windows resolves `localhost` to `::1`, and programs that listen only on IPv4 then fail inside the
tunnel. Anything with a `:` goes to ssh as you typed it, so `8080:10.0.0.5:80` works too.

A This PC → server or SOCKS tunnel gets `localhost:` in front, the address it opens on. ssh reads that as
this PC only, so a `GatewayPorts yes` in your ssh config can't open the tunnel to your network. Ports that
name their own address, such as `0.0.0.0:3000:127.0.0.1:3000`, go as typed.

## The config file

Tunnels are lines in `%APPDATA%\tether\config.ini`:

```
tunnel=web|myserver|L|3000, 5173|on
tunnel=dev||L|6111|on
```

The fields are the name, the server (a `Host` from `~/.ssh/config`), `L`, `R` or `D`, the ports, and `on`
or `off`. An empty server means the server you send to.
