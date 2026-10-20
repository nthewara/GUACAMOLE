# 01 — Background: Why FreeRDP 3.x and GFX

## The problem

Apache Guacamole provides browser-based RDP/VNC/SSH access. The RDP support is
provided by **FreeRDP**, linked into `guacd` (the Guacamole proxy daemon) via
the `libguac-client-rdp.so` plugin.

guacd releases **before 1.6.0** did not implement the **RDP Graphics Pipeline
Extension (RDPGFX)** — commonly called **GFX** or **RemoteFX Graphics
Pipeline**. Without GFX, RDP falls back to older bitmap/order-based graphics,
which is noticeably slower and lower quality, and some servers
(`gnome-remote-desktop`) refuse the connection outright
(*"Client did not advertise support for the Graphics Pipeline"*).

## What GFX gives you

guacd **1.6.0+** loads the `rdpgfx` channel by default, giving:

- **RDPGFX / RemoteFX Graphics Pipeline** streaming
- RemoteFX progressive codec
- Compatibility with modern Windows and `gnome-remote-desktop` RDP servers
  (some of which *require* GFX)

> Note: GFX support in guacd 1.6.0 works with **both FreeRDP 2.x and 3.x**
> builds — the official FreeRDP 2.11.7 image negotiates GFX too. Building
> against FreeRDP 3.x is about running a **currently maintained, security-patched**
> FreeRDP, not about unlocking GFX.

> Note: In this build we deliberately leave **H.264 (AVC420/AVC444)** disabled
> (`-DWITH_GFX_H264=OFF`). You still get the GFX channel + RemoteFX progressive,
> which is the big win. Full H.264-video RDP is a separate, heavier option.

## Guacamole version context

- **Guacamole 1.6.0** (released June 2025) is the first version to ship GFX
  support and FreeRDP 3.x compatibility in source. It is still the latest
  release as of October 2026; **1.6.1** is in progress on `staging/1.6.1`.
- The **official `guacamole/guacd` Docker image still ships FreeRDP 2.11.7**
  (both `1.6.0` and the February 2026 `latest` rebuild), on Alpine 3.18.
- guacd **1.6.0 cannot be used with FreeRDP ≥ 3.31.0** — it crashes on every
  RDP connection (GUACAMOLE-2328, fixed on `staging/1.6.1`). This is why the
  Dockerfile pins a `staging/1.6.1` commit.

Therefore, to run a current FreeRDP 3.x you must **build guacd from source**.
There is no official prebuilt image that does this.

## Important: the server side matters too

GFX only engages if **both** ends support it:

1. **Client side (this repo):** guacd 1.6.0+ (here: built against FreeRDP 3.x) ✅
2. **Server side:** the RDP server must offer GFX. Modern Windows does natively.
   On Linux, use **`gnome-remote-desktop`** (GNOME 42+), not old `xrdp`, which
   has weak/limited GFX support.

See [`06-gfx-and-servers.md`](06-gfx-and-servers.md) for details.
