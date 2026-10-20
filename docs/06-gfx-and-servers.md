# 06 — Enabling GFX + RDP server requirements

## GFX is enabled by default

In Guacamole, GFX is **opt-out**, not opt-in. You do **not** need to turn
anything on. From the Guacamole manual:

> The GFX extension is enabled by default, as most common RDP servers support
> it, and a few actually require it.

- The controlling RDP connection parameter is **`disable-gfx`**.
- In the web admin UI, the checkbox is under the **Performance** section of the
  connection editor (not "Display"), labelled exactly:
  **"Disable graphics protocol extensions (GFX):"**
- **Leave it unchecked = GFX enabled.** There is no "enable GFX" toggle because
  it is already on by default. The checkbox only exists to *disable* GFX for
  problem servers.

If you set nothing, GFX is active on every connection (and falls back gracefully
if a server does not support it).

> Reminder: GFX needs guacd **1.6.0 or newer**. It engages with both FreeRDP
> 3.x builds (this repo) and FreeRDP 2.x builds (the official image). On guacd
> 1.5.x and older the GFX channel does not exist.

## The server side must support GFX too

GFX is negotiated between client and server. It only helps if the **RDP server**
offers it:

- **Windows** — supports GFX natively (modern versions). Nothing to do.
- **Linux** — use **`gnome-remote-desktop`** (GNOME 42+), which handles
  GFX/RemoteFX well. Avoid old **`xrdp`**, which has weak/limited GFX support —
  with xrdp you may not see the benefit regardless of the client.

## Linux (Ubuntu/GNOME) RDP notes

### "Desktop Sharing" vs "Remote Login"
`gnome-remote-desktop` offers two modes:

- **Desktop Sharing** — mirrors the *existing logged-in session*. Downside: if
  the machine's screen **locks**, the shared session is suspended and you often
  **cannot RDP in** until it's unlocked at the console.
- **Remote Login** (GNOME 42+, system-level) — spins up a **new virtual session**
  on each connect, independent of the local console/lock state. This is the
  right choice for a headless devbox: **lock-independent** and GFX-capable.

Switch to **Remote Login** (Settings → System → Remote Desktop → Remote Login,
in newer Ubuntu) if "locked machine = cannot RDP" is a problem.

Caveats with Remote Login:
- It is a **separate session** from the physical console (you won't see the local
  screen).
- It may bind a **different port** and/or use **its own configured RDP
  credentials** than Desktop Sharing did — make sure your Guacamole connection's
  hostname/port/username/password match what Remote Login is configured for.

## How to confirm GFX is negotiating

Watch `docker logs -f guacd` during a connection. With a GFX-capable server the
pipeline negotiates automatically. Run guacd with `-L debug` to see it
explicitly:

```
DEBUG:  Loading Dynamic Virtual Channel rdpgfx
DEBUG:  RDPGFX channel will be used for the RDP Graphics Pipeline Extension.
```

On the Ubuntu side, `journalctl -u gnome-remote-desktop` (Remote Login) or
`journalctl --user -u gnome-remote-desktop-headless` shows
`[RDP.RDPGFX] CapsAdvertise: Accepting capability set with version RDPGFX_CAPVERSION_107`.

With **Remote Login**, the guacd debug log also shows the handover from the
login screen to your session as a server redirection
(`redirFlags: ...LB_REDIRECTION_GUID...`), followed by GFX being negotiated
again on the new session. That is expected.

If a connection is unusually slow or falls back, check:

1. guacd is 1.6.0+ (this repo's FreeRDP 3.x build — see `05-verify-troubleshoot.md`).
2. `disable-gfx` is **not** set on the connection.
3. The RDP server actually supports GFX (Windows, or `gnome-remote-desktop`).

## Tested against

October 2026, image `guacd:1.6.1-staging-freerdp3.32` (guacd `staging/1.6.1` @
`59fa92b`, FreeRDP 3.32.2, Debian 13):

| RDP server | Mode | Result |
|------------|------|--------|
| Ubuntu 26.04.1 LTS, gnome-remote-desktop 50.2 | Headless session | ✅ connects, GFX negotiated |
| Ubuntu 26.04.1 LTS, gnome-remote-desktop 50.2 | **Remote Login** (GDM greeter → user session via server redirection) | ✅ login + handover, GFX negotiated |
| xrdp 0.10.1 (Debian 13) | Xorg session | ✅ connects, GFX negotiated |

Known gap: no audio from gnome-remote-desktop (see
[`05-verify-troubleshoot.md`](05-verify-troubleshoot.md)).
