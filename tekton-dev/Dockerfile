# --- Stage 1: Build the Static Site ---
# Lock in the exact Node.js v22.14.0 requirement using a Debian base for glibc compatibility
FROM node:22.14.0-bookworm-slim AS builder

USER root
WORKDIR /app

# 1. Install Python3, pip, git (Bookworm provides git 2.39+, satisfying the 1.8.5+ requirement), and curl
RUN apt-get update && apt-get install -y \
    python3 \
    python3-pip \
    python3-venv \
    git \
    curl \
    ca-certificates \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

# 2. Lock in the exact NPM version requirement
RUN npm install -g npm@10.9.2

# 3. Lock in the exact Hugo Extended v0.107.0 requirement
ENV HUGO_VERSION=0.107.0
RUN curl -L "https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_Linux-64bit.tar.gz" \
    | tar -xz -C /usr/local/bin hugo

# Prevent Git security warnings when running as root
RUN git config --global safe.directory '*'

# Copy the Tekton website source code
COPY tekton-website/ .

# 4. Execute the Tekton Sync Script
# This reads config.yaml, uses Git to clone the sub-projects, and extracts the markdown files.
RUN python3 -m venv .venv && \
    . .venv/bin/activate && \
    pip install -r requirements.txt && \
    ./sync/sync.py

# 5. Install Node.js dependencies
RUN npm ci

# 6. Build the site for production using root-relative paths
RUN hugo --minify --environment production --baseURL "/" -d /app/public

# --- Stage 2: Serve the Offline Site ---
FROM nginx:alpine

COPY --from=builder /app/public /usr/share/nginx/html
COPY default.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]