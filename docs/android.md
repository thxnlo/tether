# tether for Android

Share a screenshot, photo, document, or text from your phone and it lands on
your Windows clipboard, ready to paste into Claude Code or Codex on the
server: **Share → tether → `Ctrl+V`.** Works on Wi-Fi and mobile data; no
Bluetooth, no VPN, nothing running on the phone between shares.

## How it works

The phone connects straight to the server's sshd, streams the files to
`/tmp/phone-in/<phone name>/`, and drops their paths into tether's receive
folder (`/tmp/clip-out`). tether's *Receive from server* puts them on the
Windows clipboard, the same way it delivers anything else from the server.

Each phone has its own key, and on the server that key can run exactly one
thing: `phone-drop`, which only stores files (200 MB per file, 2 GB in total,
deleted after a day) or up to 64 KB of text. No shell, no port forwarding.
The phone also pins the server's host key when it pairs and refuses to talk
to anything else — for this SSH path. By default a share instead goes over
HTTPS through Cloudflare, which isn't pinned and whose edge can read the
traffic; see [Fast path](#fast-path) below.

The server also knows whether a PC actually picked the share up: `phone-drop`
waits briefly for tether's *Receive from server* to fetch what it stored, so
the phone's result says **"On &lt;PC&gt;'s clipboard"** rather than just
"sent". See [Use](#use).

What it does not stop: any app on the phone can hand tether a file (Android
lets apps send it an explicit share), and a stolen key or token can upload
anything until you revoke it (see [Unpair and Replace](#unpair-and-replace)).
Either way the file's path lands on your Windows clipboard like any share,
so check what you paste into Claude.

## Server setup (once)

1. Get the server part from the [latest release](https://github.com/thxnlo/tether/releases/latest)
   and install `phone-drop`:
   ```
   curl -fsSL https://github.com/thxnlo/tether/releases/latest/download/tether-server.tar.gz | tar xz
   install -D -m 755 tether-server/phone-drop ~/bin/phone-drop
   ```
2. `~/.config/tether/phone-drop.env` with the address the phone should use:
   ```
   PUBLIC_HOST=your.server.example
   PUBLIC_PORT=22
   ```
   Optional: `INBOX`, `OUTBOX` (must match tether's receive folder setting),
   `MAX_FILE`, `MAX_TOTAL`, `TTL_MIN`, `LOCK_WAIT`, `ACK_WAIT`, `STATE_DIR`
   (defaults to `~/.config/tether/state`; holds one file per phone, its
   mtime is that phone's last upload, read by `phone-drop list`).
3. `sudo apt install qrencode` for the pairing QR code.
4. The server's sshd must be reachable from the phone and accept keys, and
   should have `ClientAliveInterval 15` set (a dropped upload is otherwise
   only noticed after a couple of hours, not seconds).

On Windows: tether running with *Receive from server* on and this server as
the target.

Run `phone-drop doctor` any time to check all of the above at once — it
prints one line per item (✓, ✗, or – for the optional fast path when it isn't set up), with the exact fix for anything missing:

```
✓ PUBLIC_HOST is set: your.server.example:22
✓ phones can reach your.server.example:22
✓ qrencode is installed
✗ sshd has no ClientAliveInterval — fix: add "ClientAliveInterval 15" to sshd_config so a dropped upload is noticed in about 45 s, not hours
✓ /tmp/phone-in is private
✓ /tmp/clip-out is private
✓ fast path answers at https://drop.example.com
✓ a PC is receiving: THX-PC
```
`phone-drop pair` runs this itself (quietly) first and warns on stderr if
something's missing, but pairing still proceeds — only a missing
`PUBLIC_HOST` stops it outright.

## Get the APK

On the phone, download `tether.apk` from the
[latest release](https://github.com/thxnlo/tether/releases/latest) and open it.
Android asks once to allow installs from your browser or file manager.
`SHA256SUMS` in the same release lists its checksum.

Updates: install a newer `tether.apk` over the old one; the pairing stays.
Or let [Obtainium](https://github.com/ImranR98/Obtainium) watch
`https://github.com/thxnlo/tether` and update it for you.

## Pair a phone

1. Install the APK on the phone.
2. Get the pairing code:
   - **From the Windows tray (recommended):** right-click the tether icon →
     **Pair a phone…**. Type a name (or leave it blank for `phone`,
     `phone-2`, and so on) and click **Show pairing code**. tether opens its
     own ssh connection to the server (the same key it uses for everything
     else) to run `phone-drop pair <name> --png`, shows the returned QR code
     right there with a countdown ("Code works for 9:58"), and opens a
     second, long-lived ssh connection that polls `phone-drop list` every
     3 s — one login kept open, not a fresh one each poll — until the phone
     shows as paired. So there's no server terminal to open at all.
     Re-pairing an already-paired name asks first, since it replaces that
     phone's key.
   - **From a terminal on the server, instead:** run the command tether
     shows on its not-paired screen — `phone-drop pair <name>` — with a
     **Copy command** button to copy it exactly. `<name>` is suggested from
     the phone's own model name (for example a Galaxy S23 suggests
     `galaxy-s23`); running `phone-drop pair` yourself with no name picks
     `phone`, `phone-2`, and so on instead. Any name is fine as long as it's
     `a-z`, `0-9`, `-`. (`phone-drop pair <name> --png` is the same command in
     a machine-readable form — `LINK`/`EXPIRES_IN`/`QRPNG` lines instead of a
     terminal QR code — which is what the tray runs above.)
3. Scan the QR code with the camera and open the link, or — if the camera
   only offers to copy the text — copy it and open tether: while the phone is
   not paired yet, it picks the link up from the clipboard. Either way this
   only fills in the field and shows "Pairing code for &lt;name&gt;
   (&lt;user&gt;@&lt;host&gt;:&lt;port&gt;) as &lt;device&gt; ✓" — the link itself, including its
   one-time key, is never shown on screen. Nothing happens until you tap
   **Pair**. The code works once, for 10 minutes.

To **replace** a pairing, open a new pairing link while the phone is paired:
run `phone-drop pair <name>` on the server, scan the QR code and open the
link. tether asks "Replace pairing with …?" first. The paired screen has no
field to paste a link into; to pair from a copied link instead, Unpair first
(a pairing code already waiting on the server survives Unpair), then open
tether, which picks the link up from the clipboard.

Pairing the same name again on the same server replaces the old key, so
there is nothing to revoke. Replacing a pairing with another server or
another name, or unpairing, removes the old key's access on that server
automatically — see [Unpair and Replace](#unpair-and-replace). `phone-drop
list` shows every paired phone, its last upload, whether it has the fast
path, and flags one idle 45+ days; `phone-drop revoke <name>` removes a
phone by hand if you'd rather not use Unpair on the phone itself.

## Use

Share → tether (after the first use it also sits in the share sheet's top
row). Nothing to open, nothing to confirm unless the batch is over 20 MB.
The first share ever also asks, in one line, whether to show notifications
("tether shows when a share lands on your PC, and why if it doesn't.") —
answered either way, it won't ask again; results still show in **Recent
shares** on tether's own screen even if you said no.

- **Files:** several files arrive as one clipboard entry with all the
  paths. A file that can't be read or is refused is skipped — the rest
  still upload and announce, so one bad file doesn't lose the batch.
- **Text and links:** share text from any app (a URL from Chrome, an error
  message copied from a log) the same way. It lands on the PC clipboard as
  the text itself, not as a path. Text over 64 KB is refused on the phone
  before it's sent.
- **Cancel:** the ongoing "Sending…" notification has a **Cancel** action.
  It stops the batch at once — the file mid-transfer is not finished or
  retried, and the server removes its partial copy. The files that had
  already uploaded stay on the server but are never sent on to your PC, so
  the result says "Cancelled — K of M uploaded to &lt;server&gt;, not sent to your
  PC", or "Cancelled — nothing sent". The server deletes those uploads after
  a day.
- **The result** always says where the file actually ended up, not just
  that it left the phone:
  - **On &lt;PC&gt;'s clipboard: N files** (or **Text is on &lt;PC&gt;'s
    clipboard**) — the PC that has *Receive from server* on took it within
    a few seconds; Ctrl+V works now.
  - **Uploaded N files to &lt;server&gt; — &lt;PC&gt; hasn't picked them up yet**
    (or **Text is on &lt;server&gt; — &lt;PC&gt; hasn't picked it up yet**) — that PC
    has *Receive from server* on but didn't take it within the few seconds
    the server waits (a slow link, a PC just waking from sleep). tether adds
    that they land there when it does; there is nothing to do.
  - **Uploaded N files to &lt;server&gt; — no PC is receiving** — the file is
    stored on the server, but no PC has *Receive from server* on. tether
    adds "Open tether on your PC and turn on Receive from server." The file
    still waits there; it arrives (into the PC's *Recent* menu, not
    silently onto the clipboard, if it's been a while) once a PC turns
    Receive on.
  - A **busy** server or a **dropped connection mid-file** is retried by
    the phone itself, up to three times (2 s, 5 s, 10 s) — you only see a
    result if every retry failed.
- **Recent shares:** tether's own screen (once paired) lists the last 20
  shares — time, a ✓ or ✗, and the same result text — under a **Recent
  shares** heading, so a result you missed (the notification timed out, or
  notifications are off) is never lost. The line above it, **Last share**,
  always shows the most recent one.

## Unpair and Replace

**Unpair** (a button at the bottom of the paired screen) asks "Unpair from
&lt;name&gt;? This phone forgets the server, and its access on the server is
removed too." On confirming, it removes the pairing from the phone and, on
the server, deletes that phone's own key and fast-path token — the same
`forget` the phone-drop key itself runs, so a phone can only ever remove its
own access. A pairing code waiting on the server for that name stays: it's
yours, not the phone's, so a re-pair you've already started still works. If
the server can't be reached, the phone still unpairs locally and shows the
manual fallback: "Unpaired on this phone. Removing its access
on &lt;name&gt; didn't work (&lt;reason&gt;) — on the server, run: phone-drop revoke
&lt;device&gt;."

**Replace** (pairing again, over an existing pairing) does the same for the
*old* server or name once the new pairing succeeds: it removes the old key
there, or — if that server can't be reached — shows the same manual
`phone-drop revoke` command. Pairing the same name on the *same* server is
not a replace: `enroll` already swapped the key, so there is nothing left to
remove on the old side.

## Fast path

Phones paired since the fast path shipped reach the server over HTTPS
through Cloudflare instead of SSH, because a plain TCP+SSH handshake costs
several round trips that HTTPS to a nearby Cloudflare edge does not. SSH
stays as the fallback and as the end-to-end option.

### Route

The app sends a batch over HTTPS by default, and falls back to SSH when: the
fast path switch is off, the target has no web URL, the target has no token,
any file's size is unknown, or any file is bigger than 95 MB (Cloudflare's
free plan caps a request at 100 MB; 95 MB leaves headroom for headers). The
web URL and the token are independent — see Pairing below. If an HTTPS
attempt fails with a network problem, the rest of that batch's retries move
to SSH right away, with no wait and without resending files that already
landed. A busy server (phone-drop's exit 5, which the receiver answers as a
503 marked `Tether-Busy: 1`) is retried on the same route instead — busy is
the server, not the route. A 503 without that mark comes from something in
front of the receiver (Cloudflare, the proxy) and counts as a network
problem: the batch moves to SSH. A rejected token, or an SSH login or
host-key failure, ends the whole batch at once instead — there is no
fallback for those, so a revoked or stale phone is told plainly that it
isn't paired rather than quietly trying the other path.

### Privacy

Cloudflare terminates TLS at its edge, so — like any other site behind its
orange cloud — it can read the file name and bytes while they pass through.
SSH stays end to end. The **Fast path via Cloudflare** switch on the pairing
screen (labelled "Cloudflare can read what passes; off = SSH only") turns
Cloudflare off entirely; every share then goes over SSH. The switch keeps
its setting even through **Unpair**.

### Server setup

- Add `WEB_URL=https://drop.example.com` to `~/.config/tether/phone-drop.env`
  (must look like `https://host[:port]`); `phone-drop pair` then adds `w=` to
  the pairing link.
- Install the receiver: `install -D -m 755 tether-server/phone-drop-web ~/bin/phone-drop-web`,
  copy `tether-server/deploy/phone-drop-web.service` to `~/.config/systemd/user/`,
  and set `LISTEN=<address>:13291` in `~/.config/tether/phone-drop-web.env`
  (the address phone-drop-web listens on — not public; a reverse proxy is the
  only way in). Then `loginctl enable-linger` and
  `systemctl --user daemon-reload && systemctl --user enable --now phone-drop-web`.
- Reverse proxy: `tether-server/deploy/nginx-drop.conf` is a template — replace
  `@HOST@` with your Cloudflare-proxied hostname (e.g. `drop.example.com`)
  and `@UPSTREAM@` with phone-drop-web's `host:port`, get a certbot
  certificate for `@HOST@`, and create a proxied (orange-cloud) DNS `A`
  record for it on the Cloudflare zone.
- `phone-drop doctor` checks the fast-path receiver too, once `WEB_URL` is set.

### Pairing

The web URL and the token are independent: `enroll` always hands the phone a
token alongside its SSH key, whatever `WEB_URL` is; only `phone-drop pair`
puts `w=` in the link, and only when `WEB_URL` is set. So a phone paired
while the server had no `WEB_URL` holds a token but no web URL, and only a
phone paired before the fast path shipped at all has neither. Either way it
shows `Route: SSH only` — pair again once `WEB_URL` is set to pick up the
fast path.

### Warm window

After a batch, the app keeps whichever connection it used (HTTPS or SSH)
open for 3 minutes of elapsed real time — counted through screen-off and
sleep, not just while the phone is awake — showing a notification titled
"tether" with the text "Ready — closes in 3 min". A share within that
window, over the same route and the same pairing, skips connecting; a batch
that fell back to SSH keeps an SSH connection, so a later share over a
different route still connects fresh. Re-pairing, even under the same name,
always starts a fresh connection. Cancelling a batch also drops its warm
connection, since the drop it was using was just closed.

`phone-drop revoke` (or Unpair's own `forget`) cuts the fast path at once:
the next HTTPS request with that phone's token gets a 401, which the app
reports the same way as a revoked SSH key: "This phone isn't paired with the
server anymore. Pair again from tether's screen." An already-open SSH
connection is different — sshd only checks `authorized_keys` at login, so a
kept, already-authenticated connection keeps working until that phone has
been idle for the warm window's 3 minutes.

## Troubleshooting

- **"The server's identity changed. If you reinstalled it, pair again. If
  you didn't, don't send anything."**: the server's key is not the one the
  phone paired with. If you did not reinstall the server, do not re-pair:
  something is between you and it.
- **"This phone isn't paired with the server anymore. Pair again from
  tether's screen."**: the key or token was revoked (Unpair, Replace, or a
  manual `phone-drop revoke`) or the pairing code expired before it was
  used — pair again.
- **"Route: SSH only — pair again to get the fast path"**: the phone is
  missing its web URL, its token, or both. Paired while the server had no
  `WEB_URL`: it has a token but no web URL. Paired before the fast path
  shipped at all: it has neither. Either way, `phone-drop pair <name>` and
  pair again once `WEB_URL` is set.
- **Where the receiver logs**: `journalctl --user -u phone-drop-web` on the
  server — one line per request (device, URL path, name, bytes, status,
  milliseconds).
- **"… — &lt;PC&gt; hasn't picked it up yet"**: that PC holds *Receive from
  server* but didn't fetch the file within the server's few-second wait. It
  lands there when it does. If it never does, the PC may have gone to sleep
  holding Receive: waking it (or restarting tether there) picks the file
  up, into its *Recent* menu once it's been a while.
- **"Uploaded … to &lt;server&gt; — no PC is receiving"**: tether is not
  running on any PC, or *Receive from server* is off everywhere. The files
  wait in the receive folder and arrive — into that PC's *Recent* menu, not
  the clipboard, once it's been a while — the moment a PC turns Receive on.
- **"The server can't store it right now (inbox full (max 2G))"**:
  `rm -rf /tmp/phone-in/*` on the server, or raise `MAX_TOTAL`.
- **"The server was still finishing an earlier upload. Share again in a
  minute."**: this is phone-drop's one-upload-per-phone lock; the app
  already retried it three times (2 s, 5 s, 10 s) before showing this. It
  clears once sshd notices the previous, usually stalled, connection is
  gone — under a minute with `ClientAliveInterval 15` in sshd_config (see
  `phone-drop doctor`), up to about two hours without it.
- **"The server can't store it right now (…is not a private directory
  of…)"**: another local user created `/tmp/phone-in` or `/tmp/clip-out`
  first (or it is a symlink). Remove it, or point `INBOX` / `OUTBOX`
  somewhere else.
- **A result with a file named on its own line, such as "&lt;name&gt;: Too
  large for the server (max 200M)"**: that one file was refused; the rest
  of the batch still uploaded (and announced) — share the large file on
  its own if you need it too.
- **"Cancelled — K of M uploaded to &lt;server&gt;, not sent to your PC"** (or
  **"Cancelled — nothing sent"**): Cancel was tapped on the notification.
  The K uploads never reach a PC; share the batch again.
- **A phone you don't remember pairing, or one you want gone**: `phone-drop
  list` on the server shows every paired phone and its last upload;
  `phone-drop revoke <name>` removes one without needing the phone at all.
