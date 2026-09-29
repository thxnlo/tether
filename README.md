# tether

[![release](https://img.shields.io/github/v/release/thxnlo/tether?color=9cc3f5&labelColor=1a1b41)](https://github.com/thxnlo/tether/releases/latest)
[![license](https://img.shields.io/badge/license-MIT-c6e85c?labelColor=1a1b41)](LICENSE)

Keep your Windows machine tethered to your SSH hosts: paste screenshots
and files into Claude Code or Codex running over SSH, keep port forwards
alive, share your local Android devices with the server, receive files
back, and drop files in from your phone. *(formerly copy-to-ssh)*

The problem: Claude Code runs on your server, your clipboard is on your
Windows machine, and Ctrl+V cannot move an image between them. Terminals
only paste text. tether runs in the tray and bridges the two — when a
screenshot or file is copied, it uploads it over SSH (streamed into
`ssh "cat > file"`, nothing to install on the server) and puts the
remote path on your clipboard, ready to paste into any SSH window.

<p align="center"><img src="demo.svg" alt="Animated demo: screenshot, click the tray icon, paste the remote path into Claude Code" width="720"></p>

## Install

The package is on winget, which also skips the SmartScreen warning below:

```powershell
winget install thxnlo.tether
```

(`winget upgrade thxnlo.tether` after a release.) Or grab `tether.exe` from
the [releases page](https://github.com/thxnlo/tether/releases/latest) — the
same release has `tether.apk` and `tether-server.tar.gz` for the phone app
(see [From your Android phone](#from-your-android-phone)).

Run the exe and it appears in the tray. Tick *Start at login* in Settings
and it starts with Windows.

The downloaded exe is unsigned, so SmartScreen shows an "unknown publisher"
warning the first time: click "More info", then "Run anyway". To skip the
dialog without winget, install from PowerShell instead:

```powershell
irm https://github.com/thxnlo/tether/releases/latest/download/tether.exe -OutFile "$env:LOCALAPPDATA\tether.exe"
Unblock-File "$env:LOCALAPPDATA\tether.exe"
& "$env:LOCALAPPDATA\tether.exe"
```

## Quick start

1. `Win+Shift+S` as usual
2. The tray icon gets a blue dot — click it (or press `Ctrl+Alt+V`)
3. Green flash → `Ctrl+V` in the SSH window: Windows Terminal, VS Code, Cursor, anything

## Tray features

Right-click the tray icon: the top of the menu spells out your state in
words — what you're sending to, whether receiving is on, tunnels, ADB —
rather than a row of glyphs.

- **Send clipboard now** sends whatever's on the clipboard right away
  (same as double-clicking the icon, or `Ctrl+Alt+V`).
- **Send to server** lists your hosts, read live from `~/.ssh/config`;
  click one to make it the current target. **Also send copies to ▸**
  broadcasts every upload to extra hosts too, without changing which one
  owns the clipboard path.
- **Sending** picks the mode: *Ask first* (default), *Auto-send every
  capture*, or *Paused*.
- **Recent** lists the last 10 sends and arrivals; click one to copy it
  back to the clipboard without re-uploading it.
- **Copy URL** (or right-click a tunnel) copies `http://localhost:<port>`
  for a running tunnel.
- **Pair a phone…** opens a small window with the phone's pairing QR code
  right from the tray — see [From your Android
  phone](#from-your-android-phone).

The tray icon's overlay glyph is the status at a glance: a blue **dot**
means a capture is waiting (click to send), a lime **check** means it
just sent and the path is on the clipboard, an amber **triangle** means a
tunnel, ADB, or the receive link is reconnecting, and a red **square**
means the last upload failed (click to retry). Hover the icon for the
same state in words, such as "sent 12 s ago → myserver". Toasts appear
only for failures, received files, and the first run.

- Copy a **file** in Explorer and it uploads the same way (200 MB cap).
- **Tunnels**: *Tunnels → Add tunnel...* — pick host, type port, done;
  supervised with keep-alive, auto-reconnect and instant restart after
  sleep. `L`/`R`/SOCKS supported. [Details](docs/tunnels.md)
- **ADB**: *Tunnels → Share ADB devices* makes your phones visible to
  `adb` on the server; wireless devices reconnect automatically.
  [Details](docs/adb.md)
- **Receive**: tick *Receive from server* and anything dropped in the
  receive folder on the host (default `/tmp/clip-out`) lands on your
  clipboard, or in Downloads for anything that isn't an image or short
  text. Only one PC receives per server at a time; a file that waited
  there for more than about a minute lands in **Recent** and Downloads
  instead — never straight onto the clipboard.

## Set up SSH for tether

tether needs a key login to the server: no password prompt on any
upload. If `ssh <server>` already logs in without asking for anything,
you're done — otherwise:

1. On Windows, in PowerShell: `ssh-keygen -t ed25519` (accept the
   defaults, or set a passphrase and add the key to `ssh-agent`).
2. Copy the public key it printed
   (`type $env:USERPROFILE\.ssh\id_ed25519.pub`) to the end of
   `~/.ssh/authorized_keys` **on the server**, for example for user
   `alice`:
   ```
   type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh alice@example.test "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
   ```
3. Add an entry to `~/.ssh/config` on Windows
   (`%USERPROFILE%\.ssh\config`) so tether — and everything else — can
   call the server by a short name:
   ```
   Host myserver
       HostName example.test
       Port 2222
       User alice
       IdentityFile ~/.ssh/id_ed25519
   ```
4. Check it: `ssh myserver true` must print nothing and never ask for a
   password or passphrase. Pick `myserver` (or whichever `Host` you
   used) in tether's *Send to server* menu.

## From your Android phone

Share a screenshot, photo, or file from any app to *tether* and its path
lands on your Windows clipboard a few seconds later — the phone's result
says **"On &lt;PC&gt;'s clipboard"** only once a PC has actually picked it
up.

You need:

- `tether.apk` from the [latest release](https://github.com/thxnlo/tether/releases/latest)
- `phone-drop`, from `tether-server.tar.gz` in the same release, installed
  on the server
- tether running on the PC with *Receive from server* turned on

Pair the phone from the tray's **Pair a phone…** — no server terminal
needed. Full setup and troubleshooting: [docs/android.md](docs/android.md).

## Requirements

Windows 10 or 11 with the built-in OpenSSH client, and key-based auth
to the target host.

## Limitations

Uploads go to the host selected in the tray (plus broadcast hosts). If
you SSH into several machines or containers, pick the one your session
actually runs on. `adb` must be on PATH for device sharing.

---

© 2026 [thxnlo](https://github.com/thxnlo) — MIT license. This repo
hosts releases and documentation; the source is developed privately.
