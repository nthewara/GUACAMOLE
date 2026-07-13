# 04 — Security hardening

Security was a primary goal of this build. Measures applied:

## Image hardening

- **Pinned FreeRDP version** (`ARG FREERDP_VERSION=3.28.0`) — always build a
  known, current, CVE-patched release. Do **not** track a moving branch.
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

**Takeaway:** always build against the **latest** FreeRDP 3.x release, and
**rebuild + re-scan** whenever FreeRDP ships a new security release. Bump
`FREERDP_VERSION`, rebuild, re-run Trivy, redeploy.

## Vulnerability scanning gate (Trivy)

Before deploying any build, scan it and refuse to ship if there are fixable
HIGH/CRITICAL vulnerabilities:

```bash
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy:latest image --scanners vuln \
  --severity HIGH,CRITICAL --ignore-unfixed --no-progress \
  guacd:1.6.0-freerdp3.28
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
