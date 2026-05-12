# --- Stage 1: Build the Static Site ---
FROM node:22-bookworm-slim AS builder

WORKDIR /app

# 1. Install prerequisites required for native Node modules and downloading external docs
RUN apt-get update && apt-get install -y \
    git \
    curl \
    python3 \
    make \
    g++ \
    && rm -rf /var/lib/apt/lists/*

RUN git config --global --add safe.directory '*'

COPY helm-www/ .

# 2. Install dependencies using Yarn
RUN yarn install --frozen-lockfile || yarn install

# 3. Pull in the external Community documentation 
RUN yarn download-remote-community || true

# 4. --- OFFLINE PATCHING & CONFIG OVERRIDES ---
# Downgrade strict broken link errors to warnings so external typos don't crash the build
RUN sed -i -E "s/onBrokenLinks: ['\"]throw['\"]/onBrokenLinks: 'warn'/g" docusaurus.config.js || true

# 5. Build the Docusaurus site (English ONLY to save space and build time)
RUN yarn build --locale en

# 6. Strip External Dependencies (Offline Mode)
RUN find build -type f -name "*.html" -exec sed -i -E 's/<link[^>]*fonts.(googleapis|gstatic).com[^>]*>//g' {} + || true && \
    find build -type f -name "*.html" -exec sed -i -E 's/<script[^>]*google-analytics[^>]*>.*<\/script>//g' {} + || true

# --- Stage 2: Serve the Offline Site ---
FROM nginx:alpine

COPY --from=builder /app/build /usr/share/nginx/html
COPY default.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
