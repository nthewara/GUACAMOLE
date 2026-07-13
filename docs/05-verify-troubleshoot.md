# 05 — Verification & troubleshooting

## Verify FreeRDP 3.x is actually linked

```bash
docker run --rm --entrypoint sh guacd:1.6.0-freerdp3.28 -c \
  'ldconfig -p | grep -i freerdp; \
   echo ---; \
   ldd /usr/local/lib/libguac-client-rdp.so | grep -i freerdp'
```

You want to see **`libfreerdp3.so.3`** / **`libfreerdp-client3.so.3`** (the `3`
confirms FreeRDP 3.x). If you see `libfreerdp2` / `libfreerdp-client2`, the
build linked against 2.x.

## Verify all plugins resolve their libraries

```bash
docker run --rm --entrypoint sh guacd:1.6.0-freerdp3.28 -c \
  'for p in rdp vnc ssh telnet; do
     echo -n "$p: ";
     ldd /usr/local/lib/libguac-client-$p.so 2>&1 | grep "not found" || echo OK;
   done'
```

All four should say `OK`. Any `not found` line names a missing runtime library
you must add to the runtime stage.

## Watch a live connection

```bash
docker logs -f guacd
```

A healthy RDP connection logs something like:

```
INFO:  Creating new client for protocol "rdp"
INFO:  Security mode: Negotiate (ANY)
INFO:  Loading keymap "base"
INFO:  Loading keymap "en-us-qwerty"
INFO:  Accepted format: 16-bit PCM with 2 channels at 44100 Hz
INFO:  User "..." joined connection "..."
```

## Troubleshooting table

### `WARNING: Support for protocol "rdp" is not installed`
The RDP plugin failed to load — almost always a **missing runtime library**.
Run the plugin `ldd` check above; add the missing lib (commonly `libfuse3-3`)
to the runtime stage and rebuild. Symptom in the browser: connections hang at
**"Waiting for response…"** then disconnect.

### Browser shows "Connected to Guacamole. Waiting for response…" forever
The web app reached guacd, but guacd could not complete the connection. Check
`docker logs guacd` — usually the "not installed" issue above, or the target RDP
server is unreachable/refusing.

### `Cannot link to a non running container`
You used legacy `--link` and recreated the linked container. Migrate the stack
to a **user-defined network** (`guac-net`) and recreate the web app without
`--link`. Containers then resolve each other by name.

### `The JAVA_HOME environment variable is not defined correctly / openjdk-8`
The **web** container (`guacamole/guacamole`) is a stale container created from
an old image. Recreate it (don't just restart) so it boots from the current
1.6.0 image (JDK 21 at `/opt/java/openjdk`).

### `RDP server closed/refused connection: Manually logged off`
Not a Guacamole error — the RDP **server** ended the session (e.g. a normal
logoff). Nothing to fix on the Guacamole side.

### FreeRDP warns about home directory writability
Cosmetic. Ensure the non-root user has a home dir (`useradd -m -d /home/guacd`).
Connections work regardless.

### Build fails: `Could NOT find Kerberos`
FreeRDP 3.x hard-requires Kerberos dev headers. Add `libkrb5-dev` to the builder
stage.

### Build fails: `autoreconf: not found`
Add `autoconf automake libtool` to the builder stage.

### Build fails: `error: '...' is deprecated [-Werror=deprecated-declarations]`
guacamole-server 1.6.0 uses FreeRDP3 APIs deprecated in newer FreeRDP. Add
`CFLAGS="-Wno-error=deprecated-declarations -Wno-deprecated-declarations"` to
the `./configure` invocation. The calls still work at runtime.
