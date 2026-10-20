# Apache Guacamole — Custom guacd with FreeRDP 3.x + GFX

Documentation and build recipe for compiling a **security-hardened `guacd`
(Guacamole proxy daemon) against a current FreeRDP 3.x release**, with the RDP
**Graphics Pipeline (GFX / RemoteFX)** extension enabled.

> The official `guacamole/guacd` Docker image still ships **FreeRDP 2.11.7**
> (on Alpine 3.18). This repo documents how to build guacd **1.6.1 (pre-release,
> pinned `staging/1.6.1` commit) against FreeRDP 3.32.2**, how to deploy it
> alongside the standard `guacamole/guacamole` web app and a database, and the
> gotchas we hit.

## Contents

- [`guacd-freerdp3/Dockerfile`](guacd-freerdp3/Dockerfile) — the multi-stage build
- [`docs/01-background.md`](docs/01-background.md) — why FreeRDP 3.x / GFX
- [`docs/02-build.md`](docs/02-build.md) — building the image (step by step + gotchas)
- [`docs/03-deploy.md`](docs/03-deploy.md) — deploying the full stack
- [`docs/04-security.md`](docs/04-security.md) — security hardening + Trivy scanning
- [`docs/05-verify-troubleshoot.md`](docs/05-verify-troubleshoot.md) — verification & troubleshooting
- [`docs/06-gfx-and-servers.md`](docs/06-gfx-and-servers.md) — enabling GFX + RDP server requirements
- [`docker-compose.example.yml`](docker-compose.example.yml) — reference stack (sanitized)

## TL;DR

```bash
# 1. Build the custom guacd (FreeRDP 3.32.2 + GFX)
cd guacd-freerdp3
docker build -t guacd:1.6.1-staging-freerdp3.32 .

# 2. Scan it for vulnerabilities (security gate)
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy:latest image --scanners vuln \
  --severity HIGH,CRITICAL --ignore-unfixed --no-progress \
  guacd:1.6.1-staging-freerdp3.32

# 3. Deploy on a user-defined network next to the web app + DB
docker network create guac-net
docker run -d --name guacd --restart always --network guac-net \
  guacd:1.6.1-staging-freerdp3.32
# ...then the guacamole/guacamole web app + database (see docs/03-deploy.md)
```

## Key facts

| Component | Version |
|-----------|---------|
| guacamole-server (guacd) | 1.6.1 pre-release — `staging/1.6.1` @ `59fa92b` (pinned) |
| FreeRDP | 3.32.2 (pinned) |
| guacamole/guacamole (web) | 1.6.0 (`latest`) |
| Base image | `debian:trixie-slim` (Debian 13) |
| Runs as | non-root user |
| GFX / RDPGFX | **enabled by default** (opt-out via `disable-gfx`) |

> **Why a pre-release guacd?** guacd **1.6.0 crashes on every RDP connection
> when built against FreeRDP ≥ 3.31.0** (it calls `gdi_init()` too early —
> GUACAMOLE-2328). The fix is only on the upstream `staging/1.6.1` branch, so
> the Dockerfile pins a commit from it. Switch to the `1.6.1` tag once Apache
> releases it. See [`docs/02-build.md`](docs/02-build.md#7-guacd-160-crashes-with-freerdp--3310).

## Why not just use the official image?

The official image is **functional**. Tested October 2026, the official
`guacamole/guacd:1.6.0` image negotiates GFX and connects to Ubuntu 26.04 LTS
`gnome-remote-desktop`, both headless and *Remote Login* (with server
redirection). guacamole-server 1.6.0 enables GFX on FreeRDP 2.x builds too.

The reason to build your own is **security currency**:

- The official image bundles **FreeRDP 2.11.7** (April 2024). The FreeRDP 3.29
  → 3.32.2 releases alone reference 158 security advisories. The 2.x line only
  gets occasional backports, and the official image predates even the latest
  of those (2.11.8, February 2026).
- It is built on **Alpine 3.18**, which is end-of-life.
- There is still no official image built against FreeRDP 3.x (the upstream
  Dockerfile defaults to FreeRDP 2 even on `staging/1.6.1`).

This repo builds guacd from source against the latest FreeRDP 3.x on a
supported Debian base, and documents how to keep it current.

## Disclaimer

Community documentation, not affiliated with the Apache Software Foundation.
"Apache Guacamole" is a trademark of the ASF. Provided as-is; review before use
in production. Pin and update FreeRDP for security fixes.
