[tether](../README.md) · [Tunnels](tunnels.md) · [Phones over adb](adb.md) · Android

# tether for Android

Share a screenshot, photo, file or text from your phone, and it lands on your Windows clipboard, ready
to paste into Claude Code or Codex on the server: **Share → tether → Ctrl+V**. It works on Wi-Fi and
mobile data, with no Bluetooth or VPN, and nothing runs on the phone between shares.

The phone sends to your server, and tether on the PC takes it from there. Over SSH it is end to end,
with a key that can only drop files. If you set up the [fast path](#the-fast-path), shares go over
HTTPS through Cloudflare instead, which is faster but lets Cloudflare read them; one switch on the
phone turns it off.

**Contents:** [Set up](#set-up) · [Share](#share) · [What the result means](#what-the-result-means) ·
[Unpair and replace](#unpair-and-replace) · [Troubleshooting](#troubleshooting) ·
[How it works](#how-it-works) · [The fast path](#the-fast-path)

## Set up

You need tether on the PC with **Receive from server** turned on and this server picked in **Send to
server**.

### 1. On the server, once

1. Install `phone-drop` from the latest release:
   ```
   url=https://github.com/thxnlo/tether/releases/latest/download/tether-server.tar.gz
   curl -fsSL "$url" | tar xz
   install -D -m 755 tether-server/phone-drop ~/bin/phone-drop
   ```
2. Tell it the address the phone should use, in `~/.config/tether/phone-drop.env`:
   ```
   PUBLIC_HOST=your.server.example
   PUBLIC_PORT=22
   ```
3. Install `qrencode` for the pairing QR code: `sudo apt install qrencode`.
4. The phone must reach the server's sshd. Set `ClientAliveInterval 15` in `sshd_config` too, or a
   dropped upload goes unnoticed for hours.
5. Check it all with `phone-drop doctor`. It prints one line per item, ✓ or ✗, and after each ✗ the
   fix:
   ```
   ✓ PUBLIC_HOST is set: s.example:22
   ✓ phones can reach s.example:22
   ✓ qrencode is installed
   ✓ sshd has ClientAliveInterval set
   ✓ a PC is receiving: WORKPC
   ```

<details>
<summary>Settings phone-drop also reads</summary>

In the same `phone-drop.env`: `INBOX` and `OUTBOX` (`OUTBOX` must match tether's **Receive folder on
the server**), `MAX_FILE`, `MAX_TOTAL`, `TTL_MIN`, `LOCK_WAIT`, `ACK_WAIT`, and `STATE_DIR` (default
`~/.config/tether/state`, one file per phone, read by `phone-drop list`). `phone-drop pair` runs the
checks quietly first, and warns about anything missing; only a missing `PUBLIC_HOST` stops it.
</details>

### 2. On the phone

Download `tether.apk` from the [latest release](https://github.com/thxnlo/tether/releases/latest) and
open it. Android asks once to allow installs from your browser or file manager. `SHA256SUMS` in the
release lists its checksum.

### 3. Pair

1. On the PC, right-click the tether icon and choose **Pair a phone…**. Type a name, or leave it
   blank for `phone`, and select **Show pairing code**. tether shows a QR code and waits.
2. Scan the QR code with the phone's camera: it opens tether. If the camera only offers to copy the
   text, copy it and open tether, which picks the link up from the clipboard.
3. Check the line tether shows ("Pairing code for … as … ✓") and tap **Pair**.

The code works once, for 10 minutes. The link and its one-time key never show on screen.

<details>
<summary>Pair from a terminal on the server instead</summary>

Run the command tether's unpaired screen shows, `phone-drop pair <name>` (its **Copy command** button
copies it). The name is suggested from the phone's model, and may use `a-z`, `0-9` and `-`. Then scan
the QR code it prints, as above. Pairing a name that is already paired replaces that phone's key, and
asks first.
</details>

## Share

In any app, tap **Share** and choose tether. After the first time it sits in the share sheet's top row.
There is nothing to open, and nothing to confirm unless the batch is over 20 MB. tether's own screen
has **Send clipboard** and **Send a file** too.

- **Files:** several files arrive as one clipboard entry with all the paths. A file that cannot be read,
  or that the server refuses, is skipped; the rest still arrive.
- **Text and links** land on the PC's clipboard as the text itself, up to 64 KB.
- **Cancel:** the "Sending…" notification has a **Cancel** button. It stops at once. Files already
  uploaded stay on the server for a day but never reach the PC.
- The first share asks once whether tether may show notifications. Either way, every result is also on
  tether's screen under **Recent**: the last five, and **Show all** for up to 20.

## What the result means

The result says where the share ended up, not only that it left the phone:

| The phone says | Means | What to do |
|---|---|---|
| On WORKPC's clipboard: 1 file | The PC took it | Paste |
| Uploaded 1 file to devbox, not on WORKPC's clipboard yet | The PC has Receive on but was slow, or woke a moment ago | Nothing: it lands there when the PC takes it |
| Uploaded 1 file to devbox, but no PC is receiving | No PC has Receive from server on | Turn on **Receive from server** on a PC |
| Cancelled: 2 of 3 uploaded to devbox, not sent to your PC | You tapped Cancel | Share again |

A busy server or a dropped connection is retried by the phone, three times (after 2, 5 and 10 s). You
see a result only when every try failed. A file that waited on the server for more than about a minute
arrives in the PC's **Recent** menu instead of on the clipboard.

## Updates

tether looks for a new release when it starts (a share, or opening it), at most once every 20 hours,
and about once a day in the background on Wi-Fi. It downloads `tether.apk` and checks it against the
release's signed list of checksums before anything is installed.

On Android 12 and later, once tether may install apps, it installs the update by itself on Wi-Fi with
the battery not low, while nothing is being shared and tether's screen is closed. A quiet
notification then says "Updated to tether 3.2.2"; tap it for the release notes. The pairing stays. If a
background update fails three times in a row, one notification says why; tap it to try again.

Android asks instead:

- The first time: one notification, "tether 3.2.2 is ready", and an **Update** card on tether's
  screen. Tapping either downloads the update; Android asks you to allow tether to install apps (the
  update carries on when you come back), then shows its install dialog.
- On Android 10 and 11, every time, the same way.
- From 3.2.0 to 3.2.1: 3.2.0 always asks, so that update shows Android's install dialog.

**Update** on tether's screen always shows Android's dialog, so you see the install happen and can
open tether again afterwards.

## Unpair and replace

**Unpair**, at the bottom of tether's screen, asks first, then removes the pairing from the phone and
this phone's access from the server. A phone can only ever remove its own access. If the server cannot
be reached, the phone still unpairs and shows the command to run on the server:
`phone-drop revoke <name>`.

**Replace:** opening a new pairing link while paired asks "Replace pairing with …?". Once the new
pairing works, tether removes the old one's access on its server, or shows the same command if it
cannot.

On the server, `phone-drop list` shows every paired phone, its last upload and whether it has the fast
path, and flags one idle for 45 days or more. `phone-drop revoke <name>` removes a phone without needing
it.

## Troubleshooting

**"The server's identity changed. If you reinstalled it, pair again. If not, do not send anything."**
The server's key is not the one the phone paired with. If you did not reinstall the server, do not pair
again: something is between the phone and the server.

### Pairing

- **"This phone isn't paired with the server anymore. Pair again from tether's screen."** Its key was
  removed (Unpair, Replace or `phone-drop revoke`), or the pairing code ran out before you used it. Pair
  again.
- **"The fast path needs a new pairing. Until then, shares go over SSH."** The phone was paired before
  the server had the fast path. Pair again to pick it up.
- **"This pairing link is cut short or damaged."** Part of the link was lost on the way, often when it
  was copied by hand. Scan the QR code again, or copy the whole link.

### Sending

- **"… not on WORKPC's clipboard yet", and it never arrives.** The PC may have gone to sleep with Receive on.
  Wake it, or restart tether there: the file arrives in its **Recent** menu.
- **"… but no PC is receiving".** tether is not running on any PC, or Receive from server is off. The files
  wait and arrive once a PC turns it on.
- **A file named on its own line, such as "IMG_2041.jpg: Too large for the server (max 200M)".** That one
  file was refused; the rest arrived. Share it on its own if you need it.

### The server

- **"The server can't store it right now (inbox full, max 2G)".** Run `rm -rf /tmp/phone-in/*` on the
  server, or raise `MAX_TOTAL`.
- **"The server was still finishing an earlier upload. Share again in a minute."** The phone already
  tried three times. It clears once sshd notices the earlier connection is gone: under a minute with
  `ClientAliveInterval 15`, up to two hours without it.
- **"The server can't store it right now (… is not a private directory of …)".** Another user created
  `/tmp/phone-in` or `/tmp/clip-out` first, or it is a symlink. Remove it, or set `INBOX` or `OUTBOX` to
  a folder of yours.
- **The fast path's log:** `journalctl --user -u phone-drop-web` on the server, one line per request.

## How it works

The phone logs in to the server's sshd, streams the files to `/tmp/phone-in/<phone name>/`, and puts
their paths in tether's receive folder (`/tmp/clip-out`). tether's **Receive from server** puts them on
the Windows clipboard, the same way it brings back anything else from the server.

Each phone has its own key, and on the server that key can run one thing: `phone-drop`, which only
stores files (200 MB each, 2 GB in all, deleted after a day) or up to 64 KB of text. No shell and no
port forwarding. The phone also pins the server's host key when it pairs, and refuses any other.

`phone-drop` waits a few seconds for the PC to take what it stored, which is how the phone knows to say
"On WORKPC's clipboard" rather than only "sent".

What this does not stop: any app on the phone can hand tether a file, and a stolen key or token can
upload anything until you revoke it. Either way the file's path lands on your clipboard like any share,
so check what you paste into Claude.

## The fast path

With the fast path, phones reach the server over HTTPS through Cloudflare instead of SSH. It is faster
because HTTPS to a nearby Cloudflare server takes fewer round trips than an SSH login. SSH stays as the
fallback, and as the end-to-end choice.

**Privacy.** Cloudflare decrypts the traffic at its edge, so it can read the file names and contents
that pass through it. SSH stays end to end. **Fast path** under **This phone** on tether's screen turns
Cloudflare off: every share then goes over SSH. The switch keeps its setting through Unpair.

### Set it up on the server

1. Add `WEB_URL=https://drop.example.com` to `~/.config/tether/phone-drop.env`. `phone-drop pair` then
   puts it in the pairing link.
2. Install the receiver and its service:
   ```
   install -D -m 755 tether-server/phone-drop-web ~/bin/phone-drop-web
   cp tether-server/deploy/phone-drop-web.service ~/.config/systemd/user/
   ```
3. Set `LISTEN=<address>:13291` in `~/.config/tether/phone-drop-web.env`: a private address, reached
   only through the reverse proxy.
4. Start it: `loginctl enable-linger`, then
   `systemctl --user daemon-reload && systemctl --user enable --now phone-drop-web`.
5. Put a reverse proxy in front. `tether-server/deploy/nginx-drop.conf` is a template: replace `@HOST@`
   with your hostname behind Cloudflare and `@UPSTREAM@` with phone-drop-web's address, get a
   certificate for the hostname, and add a proxied DNS record for it in Cloudflare.
6. `phone-drop doctor` checks the receiver too, once `WEB_URL` is set.

A phone paired before `WEB_URL` was set sends over SSH until you pair it again.

### Which way a share goes

A share goes over HTTPS unless the **Fast path** switch is off, the phone was paired without the fast
path, a file's size is unknown, or a file is over 95 MB (Cloudflare's free plan stops at 100 MB). If
HTTPS fails with a network problem, the rest of that batch goes over SSH at once, without resending what
already landed. A busy server is retried on the same route. A refused token, or a failed SSH login,
ends the batch: the phone says plainly that it is not paired.

After a share, the phone keeps that connection open for 3 minutes, so a share soon after skips
connecting. The notification says "Connection kept open 3 min for your next share". `phone-drop revoke` cuts the
fast path at once; an SSH connection that is already open keeps working until those 3 minutes pass.
