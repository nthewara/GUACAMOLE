# 02 — Building the custom guacd image

The full recipe is in [`../guacd-freerdp3/Dockerfile`](../guacd-freerdp3/Dockerfile).
It is a **multi-stage** build:

1. **builder stage** — compiles FreeRDP 3.x and guacamole-server from source
2. **runtime stage** — a minimal, non-root image with only the shared libraries
   needed at run time (no compilers → smaller attack surface)

## Build it

```bash
cd guacd-freerdp3
docker build -t guacd:1.6.1-staging-freerdp3.32 .
```

Versions are pinned via `ARG` at the top of the Dockerfile — bump these to update:

```dockerfile
ARG FREERDP_VERSION=3.32.2
ARG GUAC_VERSION=1.6.1
ARG GUAC_REF=59fa92bfc8bd6df4f0392fe99dfc78b0a0457cc7   # staging/1.6.1
```

`GUAC_REF` accepts a **tag or a full commit SHA**. It currently points at a
commit on upstream's unreleased `staging/1.6.1` branch (see gotcha #7). When
Apache tags 1.6.1, set `GUAC_REF=1.6.1`. Both `ARG` sets are repeated in the
runtime stage (for the OCI labels), so bump them in both places.

On an 8-core host a clean build takes roughly **10–20 minutes** (FreeRDP +
guacamole-server both compile from source).

## Gotchas we hit (and how they're solved in the Dockerfile)

These are the real problems encountered building this; each is already handled
in the committed Dockerfile, documented here so you understand *why*.

### 1. FreeRDP 3.x needs more build dependencies than 2.x
FreeRDP 3.x adds hard/near-hard requirements that FreeRDP 2.x did not have:

- **Kerberos** — `libkrb5-dev` (this one is a **hard** requirement; cmake fails
  without it: *"Could NOT find Kerberos"*)
- `libjson-c-dev`, `liburiparser-dev`, `libpcsclite-dev`, `libsystemd-dev`

### 2. guacamole-server needs the autotools
`autoreconf: not found` → add **`autoconf automake libtool`** to the builder.

### 3. Deprecated-API warnings become fatal (`-Werror`)
guacamole-server calls some FreeRDP3 APIs that **newer FreeRDP 3.x has
deprecated** (e.g. `pVerifyCertificate` → `pVerifyCertificateEx`, `codecs_free`
→ `freerdp_client_codecs_free`, `pVerifyChangedCertificate`). The build treats
deprecation warnings as errors and fails.

Since **FreeRDP 3.31**, FreeRDP's own public headers (`winpr/sspi.h`) also emit
deprecation warnings just by being included. Under `-Werror` this silently
breaks guacamole-server's `./configure` feature detection, and the build then
fails with *"'freerdp' has no member named 'input'"* (GUACAMOLE-2326, still
open upstream).

**Fix:** relax just that warning during `./configure` (covers both problems,
since autoconf's compile tests also use `CFLAGS`):

```dockerfile
CFLAGS="-Wno-error=deprecated-declarations -Wno-deprecated-declarations"
```

Check the build log for `checking whether freerdp structs have a context... yes`.
A `no` there means detection broke.

The deprecated calls still **function correctly at runtime** in FreeRDP 3.32 —
they are warnings, not removals.

### 4. Missing RUNTIME libraries → "protocol rdp is not installed"
This is the subtle one. If the runtime image is missing **any** shared library
that a guacamole client plugin needs, guacd fails to `dlopen` the plugin and
logs:

```
WARNING:  Support for protocol "rdp" is not installed
```

...and every RDP connection dies instantly at *"Waiting for response…"* in the
browser. The plugin is present, but a dependency is missing.

The ones that are easy to miss:

| Library | Package (trixie) | Why it is easy to miss |
|---------|------------------|------------------------|
| `libfuse3.so.4` | `libfuse3-4` | RDP plugin dependency (was `libfuse3-3` / `.so.3` on bookworm) |
| `libicuuc.so.76` | `libicu76` | WinPR/FreeRDP link ICU. On bookworm, `libxml2` pulled ICU in implicitly; trixie's `libxml2` does not |
| `libsndio.so.7` | `libsndio7.0` | FreeRDP's client lib picks up sndio when built on trixie |

The runtime stage installs these plus the FreeRDP3 runtime libs
(`libkrb5-3 libjson-c5 liburiparser1 libpcsclite1 libgssapi-krb5-2`).

> **Debian 13 (trixie) package names:** several libraries were renamed by the
> 64-bit `time_t` transition (`libssl3t64`, `libasound2t64`, `libcups2t64`,
> `libssh2-1t64`, `libpng16-16t64`, `libpkcs11-helper1t64`,
> `libwebsockets19t64`) and FFmpeg moved to 7.1 (`libavcodec61`,
> `libavformat61`, `libavutil59`, `libswscale8`). Re-check these names whenever
> the base image changes.

**Always verify** no plugin has unresolved libraries after building:

```bash
docker run --rm --entrypoint sh guacd:1.6.1-staging-freerdp3.32 -c \
  'for p in rdp vnc ssh telnet; do
     echo -n "$p: ";
     ldd /usr/local/lib/libguac-client-$p.so 2>&1 | grep "not found" || echo OK;
   done'
```

Expected output: `rdp: OK`, `vnc: OK`, `ssh: OK`, `telnet: OK`.

### 5. Plugin install location
guacamole-server installs client plugins to `/usr/local/lib/` (as
`libguac-client-*.so` + `.la` files), **not** a `guacamole/` subdirectory. The
Dockerfile copies the whole `/usr/local/lib/` tree so guacd finds them.

### 6. Non-root user needs a home directory
Running guacd as non-root is good practice, but FreeRDP warns if the user has no
writable home (used for certificate storage). The Dockerfile creates the user
with `-m -d /home/guacd`.

### 7. guacd 1.6.0 crashes with FreeRDP ≥ 3.31.0
guacd **1.6.0** calls FreeRDP's `gdi_init()` from the `PreConnect` callback.
FreeRDP **3.31.0** moved creation of the context's cache into the core, where it
now happens *after* PreConnect. So `gdi_init()` dereferences a NULL cache, and
**every RDP connection aborts** before anything is drawn. The browser hangs at
*"Waiting for response…"*, and the guacd log ends with a backtrace through
`winpr_int_assert` → `gdi_init` → `freerdp_connect`. The build itself succeeds,
so only a real connection reveals this.

- Upstream tickets: **GUACAMOLE-2328** (fixed on `staging/1.6.1` by moving
  `gdi_init()` to `PostConnect`) and **GUACAMOLE-2342** (same crash, confirmed
  fixed by that change).
- 1.6.1 also carries other FreeRDP 3.x fixes: `AuthenticateEx` support for
  FreeRDP ≥ 3.25 (GUACAMOLE-2273), RemoteApp on FreeRDP 3 (GUACAMOLE-2072), and
  connection-process hang fixes (GUACAMOLE-2118).

**Fix:** build guacamole-server from a pinned `staging/1.6.1` commit
(`GUAC_REF`) until 1.6.1 is released. If you must stay on the 1.6.0 tag, the
newest FreeRDP you can use is **3.30.0**, which misses the 3.31–3.32.2 security
fixes. Not recommended.

## Verify the image links against FreeRDP 3.x

```bash
docker run --rm --entrypoint sh guacd:1.6.1-staging-freerdp3.32 -c \
  'ldd /usr/local/lib/libguac-client-rdp.so | grep -i freerdp'
```

You should see `libfreerdp-client3.so.3` and `libfreerdp3.so.3` (the **3**
matters — that confirms FreeRDP 3.x, not 2.x).
