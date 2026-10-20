# 04 — Security hardening

Security was a primary goal of this build. Measures applied:

## Image hardening

- **Pinned FreeRDP version** (`ARG FREERDP_VERSION=3.32.2`) — always build a
  known, current, CVE-patched release. Do **not** track a moving branch.
- **Pinned guacamole-server commit** (`ARG GUAC_REF=<sha>`) — while building
  from the unreleased `staging/1.6.1` branch, pin an exact commit, never the
  branch name, so builds are reproducible and reviewed.
- **Supported base OS** — `debian:trixie-slim` (Debian 13, current stable).
  Bookworm left regular security support in June 2026 (LTS only).
- **`apt-get upgrade` at build time** in both stages — pulls the latest Debian
  security patches into the image.
- **Multi-stage build** — the final runtime image contains **no compilers or
  build tools**, only the runtime shared libraries. Smaller attack surface.
- **Non-root user** — guacd runs as an unprivileged `guacd` user, not root.
- **Minimal runtime deps** — only the libraries the plugins actually need.

## Why pin (and keep updating) FreeRDP

FreeRDP is security-sensitive (it parses untrusted data from RDP servers). It
receives regular security advisories. For example, FreeRDP **3.28.0** patched a
batch of issues including a pre-authentication heap overflow in
`crypto_rsa_common`, a double-free in the rdpsnd channel, a heap overflow in
Opus audio decoding, and a `MonitorIds` double-free.

The pace has since accelerated: the releases from **3.29.0 to 3.32.2** (July →
October 2026) reference **158 GitHub security advisories** between them. The
3.31.0 notes say *"Distributors, please update ASAP, there are severe issues
being addressed"*. The official `guacamole/guacd` image (FreeRDP 2.11.7)
receives none of these fixes.

**Takeaway:** always build against the **latest** FreeRDP 3.x release, and
**rebuild + re-scan** whenever FreeRDP ships a new security release. Bump
`FREERDP_VERSION`, rebuild, re-run Trivy, **test a real RDP connection**
(see gotcha #7 in [`02-build.md`](02-build.md)), redeploy.

> **Trivy cannot see FreeRDP.** FreeRDP and guacamole-server are compiled from
> source into `/usr/local`, so they carry no package metadata. Trivy scans the
> Debian packages only. A clean Trivy result says nothing about FreeRDP CVEs;
> keeping `FREERDP_VERSION` current is the only control for those.

## Vulnerability scanning gate (Trivy)

Before deploying any build, scan it and refuse to ship if there are fixable
HIGH/CRITICAL vulnerabilities:

```bash
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy:latest image --scanners vuln \
  --severity HIGH,CRITICAL --ignore-unfixed --no-progress \
  guacd:1.6.1-staging-freerdp3.32
```

- `--severity HIGH,CRITICAL` — only care about serious findings
- `--ignore-unfixed` — ignore vulns with no available fix (nothing you can do)

A clean result reports **0** vulnerabilities. Re-run this on **every** rebuild.

## Operational security notes

- Keep the web app (`guacamole/guacamole`) up to date too; recreate the
  container on new releases (see deploy doc).
- Put Guacamole behind a TLS-terminating reverse proxy; do not expose plain
  HTTP :8080 to untrusted networks.
- Guacamole has a built-in brute-force ban extension (enabled by default in the
  official image) — leave it on.
- Store DB credentials via a `.env` file or Docker secrets, never in the image
  or in committed compose files.
