# Multi-stage build for Prefect server with UI

# Build arguments
ARG PYTHON_VERSION=3.10
ARG BASE_IMAGE=python:${PYTHON_VERSION}-slim
ARG BUILD_PYTHON_VERSION=3.10
ARG NODE_VERSION=20.19.0
ARG NODE_V2_VERSION=22.12.0
ARG SQLITE_VERSION=3.50.4
ARG EXTRA_PIP_PACKAGES=""
ARG PREFECT_EXTRAS=[redis,client,otel]
ARG VITE_AMPLITUDE_API_KEY=""

# Stage 1: SQLite builder - pulls pre-compiled SQLite binaries
FROM prefecthq/prefect-sqlite:${SQLITE_VERSION} AS sqlite-builder

# Stage 2: UI builder - builds V1 UI distributable using Node 20
FROM node:${NODE_VERSION}-slim AS ui-builder

WORKDIR /app

# Install git (required for build, version >= 2.47.3)
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    git \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# Create placeholder for UI build
RUN mkdir -p /app/ui/dist && \
    echo '{"version": "1.0.0"}' > /app/ui/dist/index.html

# Stage 3: UI V2 builder - builds V2 UI distributable using Node 22
FROM node:${NODE_V2_VERSION}-slim AS ui-v2-builder

ARG VITE_AMPLITUDE_API_KEY

WORKDIR /app

# Install git and chromium for arm64 builds
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    git \
    ca-certificates \
    chromium \
    && rm -rf /var/lib/apt/lists/*

# Create placeholder for UI V2 build
RUN mkdir -p /app/ui-v2/dist && \
    echo '{"version": "2.0.0"}' > /app/ui-v2/dist/index.html

ENV VITE_AMPLITUDE_API_KEY=${VITE_AMPLITUDE_API_KEY}

# Stage 4: Python builder - creates Python source distributable with UIs packaged
FROM python:${BUILD_PYTHON_VERSION}-slim AS python-builder

WORKDIR /app

# Install git (required, version >= 2.47.3)
# For Debian Bookworm, install from Trixie repository
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    ca-certificates \
    gnupg \
    && echo "deb http://deb.debian.org/debian trixie main" > /etc/apt/sources.list.d/trixie.list && \
    apt-get update && \
    apt-get install -y --no-install-recommends -t trixie git && \
    rm -rf /var/lib/apt/lists/*

# Install UV package manager
RUN pip install --no-cache-dir uv==0.6.17

# Copy built UIs from previous stages
COPY --from=ui-builder /app/ui/dist ./ui/dist
COPY --from=ui-v2-builder /app/ui-v2/dist ./ui-v2/dist

# Create a minimal Python package structure
RUN mkdir -p /app/dist

# Stage 5: Conda-based alternative base image (optional)
FROM continuumio/miniconda3:latest AS prefect-conda

# This stage provides a conda-based alternative but is not used in the final image

# Stage 6: Final image with Prefect installed
ARG BASE_IMAGE
FROM ${BASE_IMAGE} AS final

ARG EXTRA_PIP_PACKAGES
ARG PREFECT_EXTRAS
ARG SQLITE_VERSION

# Set environment variables
ENV LC_ALL=C.UTF-8
ENV LANG=C.UTF-8
ENV UV_COMPILE_BYTECODE=1
ENV UV_LINK_MODE=copy
ENV UV_SYSTEM_PYTHON=1
ENV LD_LIBRARY_PATH=/usr/local/lib

WORKDIR /opt/prefect

# Install system dependencies including curl for healthchecks and tini
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    curl \
    ca-certificates \
    tini \
    && rm -rf /var/lib/apt/lists/*

# Copy SQLite binaries from sqlite-builder
COPY --from=sqlite-builder /usr/local/lib/libsqlite3.* /usr/local/lib/
COPY --from=sqlite-builder /usr/local/bin/sqlite3 /usr/local/bin/
COPY --from=sqlite-builder /usr/local/include/sqlite3*.h /usr/local/include/

# Update library cache
RUN ldconfig

# Install UV package manager
RUN pip install --no-cache-dir uv==0.6.17

# Install Prefect with extras
RUN uv pip install --system "prefect${PREFECT_EXTRAS}" && \
    if [ -n "${EXTRA_PIP_PACKAGES}" ]; then \
        uv pip install --system ${EXTRA_PIP_PACKAGES}; \
    fi

# Copy UI artifacts from builder stages
COPY --from=ui-builder /app/ui/dist /opt/prefect/ui/dist
COPY --from=ui-v2-builder /app/ui-v2/dist /opt/prefect/ui-v2/dist

# Create entrypoint script
RUN echo '#!/bin/bash\n\
set -e\n\
exec "$@"' > /opt/prefect/entrypoint.sh && \
    chmod +x /opt/prefect/entrypoint.sh

# Set entrypoint
ENTRYPOINT ["/usr/bin/tini", "-g", "--", "/opt/prefect/entrypoint.sh"]

# Default command
CMD ["prefect", "server", "start"]
