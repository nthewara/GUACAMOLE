# ---- Security-hardened custom guacd with FreeRDP 3.x + GFX (RDPGFX / RemoteFX Graphics Pipeline) ----
#
# Why this image exists:
#   The official `guacamole/guacd` image still ships FreeRDP 2.11.7 (April 2024)
#   on an end-of-life Alpine base, so it misses years of FreeRDP security fixes.
#   This builds guacamole-server against a current, pinned FreeRDP 3.x release
#   with the RDP Graphics Pipeline (GFX / RemoteFX) enabled.
#
# Build:
#   docker build -t guacd:1.6.1-staging-freerdp3.32 .
#
# Pinned, reproducible versions via ARG (bump these for updates).

# ============================================================
# Stage 1: builder
# ============================================================
FROM debian:trixie-slim AS builder

ARG FREERDP_VERSION=3.32.2

# guacamole-server is pinned to a COMMIT on the upstream `staging/1.6.1` branch,
# not the 1.6.0 tag. guacd 1.6.0 calls gdi_init() from PreConnect, which
# crashes EVERY RDP connection with FreeRDP >= 3.31.0 (GUACAMOLE-2328 /
# GUACAMOLE-2342). The fix only exists on staging/1.6.1 (1.6.1 is unreleased).
# When 1.6.1 is tagged, switch GUAC_REF to the `1.6.1` tag.
ARG GUAC_VERSION=1.6.1
ARG GUAC_REF=59fa92bfc8bd6df4f0392fe99dfc78b0a0457cc7

ENV DEBIAN_FRONTEND=noninteractive

# Pull latest security patches, then all build dependencies.
# NOTE: FreeRDP 3.x needs several deps that 2.x did not:
#   Kerberos (libkrb5-dev) is a HARD requirement; also json-c, uriparser,
#   pcsclite, systemd. autoconf/automake/libtool are needed by guacamole-server.
RUN apt-get update && apt-get -y upgrade && apt-get install -y --no-install-recommends \
    build-essential cmake git ca-certificates pkg-config \
    autoconf automake libtool \
    libssl-dev libx11-dev libxext-dev libxcursor-dev libxdamage-dev \
    libxv-dev libxkbfile-dev libasound2-dev libcups2-dev libxml2-dev \
    libxrandr-dev libgstreamer1.0-dev libgstreamer-plugins-base1.0-dev \
    libxi-dev libjpeg-dev libpng-dev \
    libtool libwebp-dev libpulse-dev libavcodec-dev libavformat-dev \
    libavutil-dev libswscale-dev libssh2-1-dev libvncserver-dev \
    libpango1.0-dev libwebsockets-dev libtelnet-dev \
    uuid-dev libvorbis-dev libwebp-dev zlib1g-dev libsdl2-dev libsdl2-ttf-dev \
    libusb-1.0-0-dev libudev-dev libfuse3-dev libcjson-dev libpkcs11-helper1-dev \
    libkrb5-dev libjson-c-dev liburiparser-dev libpcsclite-dev libsystemd-dev \
    wget curl && rm -rf /var/lib/apt/lists/*

# --- Build FreeRDP 3.x (pinned, GFX enabled) ---
WORKDIR /src
RUN git clone --depth 1 --branch ${FREERDP_VERSION} https://github.com/FreeRDP/FreeRDP.git freerdp
WORKDIR /src/freerdp/build
RUN cmake -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_INSTALL_PREFIX=/usr/local \
    -DWITH_SERVER=OFF -DWITH_CLIENT=ON -DWITH_CLIENT_SDL=OFF \
    -DWITH_X11=ON -DWITH_GFX_H264=OFF -DWITH_SWSCALE=ON \
    -DWITH_FFMPEG=ON -DWITH_CUPS=ON -DWITH_PULSE=ON \
    -DBUILD_SHARED_LIBS=ON .. && \
    make -j$(nproc) && make install && ldconfig

# --- Build guacamole-server against FreeRDP 3.x ---
# NOTE: guacamole-server uses some FreeRDP3 APIs that newer FreeRDP 3.x has
# deprecated, and FreeRDP >= 3.31 headers emit deprecation warnings on mere
# inclusion. Under the build's -Werror those warnings would (a) fail the
# compile and (b) silently break configure's FreeRDP feature detection
# (GUACAMOLE-2326: "'freerdp' has no member named 'input'"). Relaxing that one
# warning fixes both; the deprecated calls still function correctly at runtime.
WORKDIR /src/guacamole-server
RUN git init -q && \
    git remote add origin https://github.com/apache/guacamole-server.git && \
    git fetch -q --depth 1 origin ${GUAC_REF} && \
    git checkout -q FETCH_HEAD
RUN autoreconf -fi && PKG_CONFIG_PATH=/usr/local/lib/pkgconfig \
    CFLAGS="-Wno-error=deprecated-declarations -Wno-deprecated-declarations" ./configure \
    --prefix=/usr/local --with-freerdp --disable-guaclog && \
    make -j$(nproc) && make install && ldconfig

# ============================================================
# Stage 2: runtime (minimal, non-root)
# ============================================================
FROM debian:trixie-slim

ARG FREERDP_VERSION=3.32.2
ARG GUAC_VERSION=1.6.1
ARG GUAC_REF=59fa92bfc8bd6df4f0392fe99dfc78b0a0457cc7
LABEL org.opencontainers.image.title="guacd (FreeRDP 3.x + GFX)" \
      org.opencontainers.image.version="${GUAC_VERSION}-freerdp${FREERDP_VERSION}" \
      org.opencontainers.image.revision="${GUAC_REF}" \
      org.opencontainers.image.source="https://github.com/nthewara/GUACAMOLE"

ENV DEBIAN_FRONTEND=noninteractive

# Runtime shared libraries only (no compilers => smaller attack surface).
# CRITICAL: these must cover every lib the guacamole client plugins dlopen,
# or guacd will silently report 'protocol "rdp" is not installed'. The ones
# that are easy to miss:
#   - libfuse3-4  (RDP plugin needs libfuse3.so.4; was libfuse3-3 on bookworm)
#   - libicu76    (WinPR/FreeRDP link ICU; bookworm's libxml2 pulled ICU in
#                  implicitly, trixie's does not)
#   - libsndio7.0 (FreeRDP's client lib links sndio when built on trixie)
RUN apt-get update && apt-get -y upgrade && apt-get install -y --no-install-recommends \
    libssl3t64 libx11-6 libxext6 libxcursor1 libxdamage1 libxv1 libxkbfile1 \
    libasound2t64 libcups2t64 libxml2 libpango-1.0-0 libpangocairo-1.0-0 libcairo2 \
    libwebp7 libpulse0 libavcodec61 libavformat61 libavutil59 libswscale8 \
    libssh2-1t64 libvncclient1 libtelnet2 libwebsockets19t64 libjpeg62-turbo libpng16-16t64 \
    libvorbis0a libvorbisenc2 libwebpmux3 libusb-1.0-0 libcjson1 libpkcs11-helper1t64 \
    libkrb5-3 libjson-c5 liburiparser1 libpcsclite1 libgssapi-krb5-2 libfuse3-4 \
    libicu76 libsndio7.0 \
    ca-certificates netcat-openbsd && rm -rf /var/lib/apt/lists/*

COPY --from=builder /usr/local/lib/ /usr/local/lib/
COPY --from=builder /usr/local/sbin/guacd /usr/local/sbin/guacd
COPY --from=builder /usr/local/bin/ /usr/local/bin/
RUN ldconfig

# Run as non-root. The -m/-d home dir avoids a FreeRDP warning about the
# current user's home directory not being writable (used for cert storage).
RUN groupadd -r guacd && useradd -r -g guacd -m -d /home/guacd guacd
USER guacd

EXPOSE 4822
ENTRYPOINT ["/usr/local/sbin/guacd"]
CMD ["-b", "0.0.0.0", "-L", "info", "-f"]
