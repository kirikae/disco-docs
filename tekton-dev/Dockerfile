# --- Stage 1: Build the Static Site ---
FROM node:22.14.0-bookworm-slim AS builder

USER root
WORKDIR /app

# 1. Install Python3, pip, git, and curl
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

RUN git config --global safe.directory '*'
COPY tekton-website/ .

# 4. Install Node modules using `npm install` (Fixes the EUSAGE error!)
RUN npm install

# 5. Execute the Tekton Sync Script
# Runs inside the virtual environment to pull all the markdown files from other repos
RUN python3 -m venv .venv && \
    . .venv/bin/activate && \
    pip install -r requirements.txt && \
    ./sync/sync.py

# 6. Build the site for production using root-relative paths
RUN hugo --minify --environment production --baseURL "/" -d /app/public

# --- Stage 2: Serve the Offline Site ---
FROM nginx:alpine

COPY --from=builder /app/public /usr/share/nginx/html
COPY default.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]