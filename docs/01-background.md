# 01 — Background: Why FreeRDP 3.x and GFX

## The problem

Apache Guacamole provides browser-based RDP/VNC/SSH access. The RDP support is
provided by **FreeRDP**, linked into `guacd` (the Guacamole proxy daemon) via
the `libguac-client-rdp.so` plugin.

Older/default guacd builds link against **FreeRDP 2.x**, which does **not**
implement the **RDP Graphics Pipeline Extension (RDPGFX)** — commonly called
**GFX** or **RemoteFX Graphics Pipeline**. Without GFX, RDP falls back to
older bitmap/order-based graphics, which is noticeably slower and lower quality,
especially over higher-latency links or with graphically busy desktops.

## What FreeRDP 3.x changes

Building guacd against **FreeRDP 3.x** enables the GFX channel, giving:

- **RDPGFX / RemoteFX Graphics Pipeline** streaming
- RemoteFX progressive codec
- Better handling of modern Windows and `gnome-remote-desktop` RDP servers
  (some of which *require* GFX)

> Note: In this build we deliberately leave **H.264 (AVC420/AVC444)** disabled
> (`-DWITH_GFX_H264=OFF`). You still get the GFX channel + RemoteFX progressive,
> which is the big win. Full H.264-video RDP is a separate, heavier option.

## Guacamole version context

- **Guacamole 1.6.0** (released 2025) is the first version to ship GFX support
  and FreeRDP 3.x compatibility in source.
- The **official `guacamole/guacd` Docker image still ships FreeRDP 2.x** even
  at 1.6.0, for backwards compatibility.
- FreeRDP 3.x support in guacamole-server is still marked **experimental**
  upstream at the time of writing.

Therefore, to get GFX you must **build guacd from source against FreeRDP 3.x**.
There is no official prebuilt image that does this.

## Important: the server side matters too

GFX only engages if **both** ends support it:

1. **Client side (this repo):** guacd built against FreeRDP 3.x ✅
2. **Server side:** the RDP server must offer GFX. Modern Windows does natively.
   On Linux, use **`gnome-remote-desktop`** (GNOME 42+), not old `xrdp`, which
   has weak/limited GFX support.

See [`06-gfx-and-servers.md`](06-gfx-and-servers.md) for details.
