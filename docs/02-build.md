# 02 — Building the custom guacd image

The full recipe is in [`../guacd-freerdp3/Dockerfile`](../guacd-freerdp3/Dockerfile).
It is a **multi-stage** build:

1. **builder stage** — compiles FreeRDP 3.x and guacamole-server from source
2. **runtime stage** — a minimal, non-root image with only the shared libraries
   needed at run time (no compilers → smaller attack surface)

## Build it

```bash
cd guacd-freerdp3
docker build -t guacd:1.6.0-freerdp3.28 .
```

Versions are pinned via `ARG` at the top of the Dockerfile — bump these to update:

```dockerfile
ARG FREERDP_VERSION=3.28.0
ARG GUAC_VERSION=1.6.0
```

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
guacamole-server 1.6.0 calls some FreeRDP3 APIs that **newer FreeRDP 3.x has
deprecated** (e.g. `pVerifyCertificate` → `pVerifyCertificateEx`, `codecs_free`
→ `freerdp_client_codecs_free`, `pVerifyChangedCertificate`). The build treats
deprecation warnings as errors and fails.

**Fix:** relax just that warning during `./configure`:

```dockerfile
CFLAGS="-Wno-error=deprecated-declarations -Wno-deprecated-declarations"
```

The deprecated calls still **function correctly at runtime** in FreeRDP 3.28 —
they are warnings, not removals. (As of this writing there is no upstream PR or
JIRA to clean these up, so this workaround remains necessary.)

### 4. Missing RUNTIME libraries → "protocol rdp is not installed"
This is the subtle one. If the runtime image is missing **any** shared library
that a guacamole client plugin needs, guacd fails to `dlopen` the plugin and
logs:

```
WARNING:  Support for protocol "rdp" is not installed
```

...and every RDP connection dies instantly at *"Waiting for response…"* in the
browser. The plugin is present, but a dependency is missing.

The one that is easy to miss is **`libfuse3-3`** (the RDP plugin needs
`libfuse3.so.3`). The runtime stage installs it plus the FreeRDP3 runtime libs
(`libkrb5-3 libjson-c5 liburiparser1 libpcsclite1 libgssapi-krb5-2`).

**Always verify** no plugin has unresolved libraries after building:

```bash
docker run --rm --entrypoint sh guacd:1.6.0-freerdp3.28 -c \
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

## Verify the image links against FreeRDP 3.x

```bash
docker run --rm --entrypoint sh guacd:1.6.0-freerdp3.28 -c \
  'ldd /usr/local/lib/libguac-client-rdp.so | grep -i freerdp'
```

You should see `libfreerdp-client3.so.3` and `libfreerdp3.so.3` (the **3**
matters — that confirms FreeRDP 3.x, not 2.x).
