FROM python:3.11-slim AS ffmpeg-downloader

# 8.1, not 7.1. This downloads from BtbN's ROLLING `latest` release tag, which
# only carries the versions currently being built -- today master, n8.1 and
# n9.0. n7.1 has aged out of it, so the pinned 7.1 URL returns 404 on EVERY
# architecture and this image cannot build at all. Verified: both 8.1 archives
# return 200, both 7.1 archives return 404.
#
# The coupling is the real hazard: a version pin against a rolling tag is not a
# pin, and this will break again when 8.1 ages out. Either keep this value in
# the set `latest` still publishes, or switch to a dated autobuild tag for a
# genuine pin.
ARG FFMPEG_VERSION=8.1
# TARGETARCH is supplied by buildx. The archive slug was hardcoded to linux64,
# which put x86-64 binaries on ARM64 nodes (Jetson, SPARK) -- and because the
# runtime stage below prepends /opt/ffmpeg/bin to PATH, they did not merely sit
# unused, they SHADOWED the architecture-native ffmpeg. The slug appears in both
# the archive name and the extracted directory name, so it has to be resolved
# before either is formed.
ARG TARGETARCH

ENV DEBIAN_FRONTEND=noninteractive

# Download the pinned FFmpeg 7.1 build with HDR/AV1/NVENC/VAAPI support
RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates \
    curl \
    xz-utils \
    && rm -rf /var/lib/apt/lists/*

# Fails closed on an unrecognised architecture rather than falling back to
# linux64. A silent x86 default is exactly how this reaches an ARM node and
# only surfaces at the first transcode.
RUN set -eux; \
    case "${TARGETARCH:-amd64}" in \
      amd64) FFMPEG_ARCH=linux64 ;; \
      arm64) FFMPEG_ARCH=linuxarm64 ;; \
      *) echo "Unsupported TARGETARCH for FFmpeg build: ${TARGETARCH}" >&2; exit 1 ;; \
    esac; \
    FFMPEG_SLUG="ffmpeg-n${FFMPEG_VERSION}-latest-${FFMPEG_ARCH}-gpl-${FFMPEG_VERSION}"; \
    curl -fsSL "https://github.com/BtbN/FFmpeg-Builds/releases/download/latest/${FFMPEG_SLUG}.tar.xz" -o /tmp/ffmpeg.tar.xz; \
    mkdir -p /opt; \
    tar -xJf /tmp/ffmpeg.tar.xz -C /opt; \
    mv "/opt/${FFMPEG_SLUG}" /opt/ffmpeg; \
    rm -f /tmp/ffmpeg.tar.xz; \
    # Smoke the binary in the stage that produced it, so a wrong-architecture
    # download fails the build instead of the first transcode.
    /opt/ffmpeg/bin/ffmpeg -version | head -1

FROM python:3.11-slim

ENV DEBIAN_FRONTEND=noninteractive
ENV PATH="/opt/ffmpeg/bin:${PATH}"

WORKDIR /app

# Runtime dependencies required for audio processing and GPU detection utilities
RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates \
    curl \
    libasound2 \
    libdrm2 \
    libsndfile1 \
    libva2 \
    libva-drm2 \
    libva-x11-2 \
    libvdpau1 \
    ocl-icd-libopencl1 \
    vainfo \
    && rm -rf /var/lib/apt/lists/*

COPY --from=ffmpeg-downloader /opt/ffmpeg /opt/ffmpeg

# Copy requirements first for better caching
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

# Create necessary directories
RUN mkdir -p /app/logs /app/output /app/temp

# Set permissions
RUN chmod +x /app/entrypoint.sh

EXPOSE 8001


# Health check (process-based for non-HTTP worker)
HEALTHCHECK --interval=30s --timeout=5s --start-period=15s --retries=3 \
    CMD test -f /proc/1/cmdline || exit 1
# Run as a non-root user. The hardening ratchet judges the LAST USER directive,
# and uid/gid 65532 is the PMOVES convention (see services/agent-zero and
# services/gateway-agent). Ownership is set explicitly rather than with a late
# recursive chown so the layer stays small.
RUN groupadd -r pmoves --gid=65532     && useradd -r -g pmoves --uid=65532 --home-dir=/app --shell=/sbin/nologin pmoves     && chown -R 65532:65532 /app
USER pmoves:pmoves

CMD ["./entrypoint.sh"]
