# Apache Guacamole — Custom guacd with FreeRDP 3.x + GFX

Documentation and build recipe for compiling a **security-hardened `guacd`
(Guacamole proxy daemon) against FreeRDP 3.x**, unlocking the RDP **Graphics
Pipeline (GFX / RemoteFX)** extension that the official images do not provide.

> The official `guacamole/guacd` Docker image ships **FreeRDP 2.x**, which
> lacks GFX. This repo documents how to build guacd **1.6.0 against
> FreeRDP 3.28.0** to get GFX-accelerated RDP, how to deploy it alongside the
> standard `guacamole/guacamole` web app and a database, and the gotchas we hit.

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
# 1. Build the custom guacd (FreeRDP 3.28.0 + GFX)
cd guacd-freerdp3
docker build -t guacd:1.6.0-freerdp3.28 .

# 2. Scan it for vulnerabilities (security gate)
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy:latest image --scanners vuln \
  --severity HIGH,CRITICAL --ignore-unfixed --no-progress \
  guacd:1.6.0-freerdp3.28

# 3. Deploy on a user-defined network next to the web app + DB
docker network create guac-net
docker run -d --name guacd --restart always --network guac-net \
  guacd:1.6.0-freerdp3.28
# ...then the guacamole/guacamole web app + database (see docs/03-deploy.md)
```

## Key facts

| Component | Version |
|-----------|---------|
| guacamole-server (guacd) | 1.6.0 |
| FreeRDP | 3.28.0 (pinned) |
| guacamole/guacamole (web) | 1.6.0 (`latest`) |
| Base image | `debian:bookworm-slim` |
| Runs as | non-root user |
| GFX / RDPGFX | **enabled by default** (opt-out via `disable-gfx`) |

## Why not just use the official image?

Apache Guacamole **1.6.0** added FreeRDP 3.x *support in source*, and it is the
release that shipped the GFX / RemoteFX Graphics Pipeline. **However, the
official `guacamole/guacd:1.6.0` Docker image still builds against FreeRDP 2.x
on purpose** for compatibility. There is no official prebuilt image with
FreeRDP 3.x, so to actually get GFX you must build guacd from source against
FreeRDP 3.x — which is what this repo does.

## Disclaimer

Community documentation, not affiliated with the Apache Software Foundation.
"Apache Guacamole" is a trademark of the ASF. Provided as-is; review before use
in production. Pin and update FreeRDP for security fixes.
