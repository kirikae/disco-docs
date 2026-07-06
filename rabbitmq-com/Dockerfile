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

# --- Stage 2: Serve the Offline Site ---
FROM nginx:alpine

COPY --from=builder /app/build /usr/share/nginx/html
COPY default.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
