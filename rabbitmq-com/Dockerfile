# --- Stage 1: Build the Static Site ---
FROM node:20-bookworm-slim AS builder

WORKDIR /app

# 1. Install prerequisites required for native Node modules and git-based tooling
RUN apt-get update && apt-get install -y \
    git \
    curl \
    python3 \
    make \
    g++ \
    && rm -rf /var/lib/apt/lists/*

RUN git config --global --add safe.directory '*'

# Copy the RabbitMQ website source
COPY rabbitmq-website/ .

# 2. Install dependencies (patch-package runs automatically via the postinstall hook)
RUN npm ci || npm install

# 3. --- OFFLINE PATCHING & CONFIG OVERRIDES ---
# Downgrade strict broken-link errors to warnings so external typos don't crash the build
RUN sed -i -E "s/onBrokenLinks: ['\"]throw['\"]/onBrokenLinks: 'warn'/g" docusaurus.config.js || true && \
    sed -i -E "s/onBrokenMarkdownLinks: ['\"]throw['\"]/onBrokenMarkdownLinks: 'warn'/g" docusaurus.config.js || true

# 4. Build the Docusaurus site (all versions are compiled natively into /app/build)
RUN npm run build

# 5. Strip External Dependencies (Offline Mode)
RUN find build -type f -name "*.html" -exec sed -i -E 's/<link[^>]*fonts.(googleapis|gstatic).com[^>]*>//g' {} + || true && \
    find build -type f -name "*.html" -exec sed -i -E 's/<script[^>]*google-analytics[^>]*>.*<\/script>//g' {} + || true

# The hardened nginx image serves as UID 65532, which can only read
# world-readable content. Nothing above guarantees that, so sweep the
# output: a+rX adds read for everyone and traverse on directories only,
# without marking regular files executable.
RUN chmod -R a+rX /app/build

# --- Stage 2: Serve the Offline Site ---
# Red Hat Hardened Images nginx: distroless (no shell, no package manager) and
# runs as UID 65532, so this stage listens on 8080 (see default.conf), can RUN
# nothing, and must not set a CMD -- the image's own CMD carries the flags nginx
# needs after its /usr/sbin/nginx entrypoint. See README.md for the details and
# for how to verify the signatures on the published images.
FROM registry.access.redhat.com/hi/nginx:latest

COPY --from=builder /app/build /usr/share/nginx/html
COPY default.conf /etc/nginx/conf.d/default.conf

EXPOSE 8080
