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

> Reminder: GFX only actually engages when guacd is built against **FreeRDP 3.x**
> (this repo). On a FreeRDP 2.x guacd the checkbox does nothing.

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

Watch `docker logs -f guacd` during a connection. With FreeRDP 3.x + a
GFX-capable server, the pipeline negotiates automatically. If a connection is
unusually slow or falls back, check:

1. guacd is the FreeRDP 3.x build (see `05-verify-troubleshoot.md`).
2. `disable-gfx` is **not** set on the connection.
3. The RDP server actually supports GFX (Windows, or `gnome-remote-desktop`).
