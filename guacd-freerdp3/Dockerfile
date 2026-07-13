# ---- Security-hardened custom guacd with FreeRDP 3.x + GFX (RDPGFX / RemoteFX Graphics Pipeline) ----
#
# Why this image exists:
#   The official `guacamole/guacd` image ships FreeRDP 2.x, which lacks the
#   RDP Graphics Pipeline (GFX / RemoteFX) extension. Building guacamole-server
#   against FreeRDP 3.x unlocks GFX for far better RDP performance/quality.
#
# Build:
#   docker build -t guacd:1.6.0-freerdp3.28 .
#
# Pinned, reproducible versions via ARG (bump these for updates).

# ============================================================
# Stage 1: builder
# ============================================================
FROM debian:bookworm-slim AS builder

ARG FREERDP_VERSION=3.28.0
ARG GUAC_VERSION=1.6.0

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
    libpango1.0-dev libwebsockets-dev libtelnet-dev freerdp2-x11 \
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
# NOTE: guacamole-server 1.6.0 uses some FreeRDP3 APIs that newer FreeRDP 3.x
# has deprecated (e.g. pVerifyCertificate -> ...Ex, codecs_free -> ...). Those
# are warnings-as-errors under the default build, so we relax that one warning.
# The deprecated calls still function correctly at runtime in FreeRDP 3.28.
WORKDIR /src
RUN git clone --depth 1 --branch ${GUAC_VERSION} https://github.com/apache/guacamole-server.git
WORKDIR /src/guacamole-server
RUN autoreconf -fi && PKG_CONFIG_PATH=/usr/local/lib/pkgconfig \
    CFLAGS="-Wno-error=deprecated-declarations -Wno-deprecated-declarations" ./configure \
    --prefix=/usr/local --with-freerdp --disable-guaclog && \
    make -j$(nproc) && make install && ldconfig

# ============================================================
# Stage 2: runtime (minimal, non-root)
# ============================================================
FROM debian:bookworm-slim
ENV DEBIAN_FRONTEND=noninteractive

# Runtime shared libraries only (no compilers => smaller attack surface).
# CRITICAL: these must cover every lib the guacamole client plugins dlopen,
# or guacd will silently report 'protocol "rdp" is not installed'. The one
# that is easy to miss is libfuse3-3 (the RDP plugin needs libfuse3.so.3).
RUN apt-get update && apt-get -y upgrade && apt-get install -y --no-install-recommends \
    libssl3 libx11-6 libxext6 libxcursor1 libxdamage1 libxv1 libxkbfile1 \
    libasound2 libcups2 libxml2 libpango-1.0-0 libpangocairo-1.0-0 libcairo2 \
    libwebp7 libpulse0 libavcodec59 libavformat59 libavutil57 libswscale6 \
    libssh2-1 libvncclient1 libtelnet2 libwebsockets17 libjpeg62-turbo libpng16-16 \
    libvorbis0a libvorbisenc2 libwebpmux3 libusb-1.0-0 libcjson1 libpkcs11-helper1 \
    libkrb5-3 libjson-c5 liburiparser1 libpcsclite1 libgssapi-krb5-2 libfuse3-3 \
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
