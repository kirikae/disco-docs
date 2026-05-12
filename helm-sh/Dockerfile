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

# Copy the Helm website source
COPY helm-www/ .

# 2. Install dependencies using Yarn (Helm's preferred package manager for Docusaurus)
RUN yarn install --frozen-lockfile || yarn install

# 3. Pull in the external Community documentation 
# Helm explicitly stores its community docs in a separate GitHub repository. 
# This native script downloads them locally before the build begins.
RUN yarn download-remote-community || true

# 4. Build the Docusaurus site
# Docusaurus natively compiles all historical versions and outputs purely static files.
RUN yarn build

# 5. Strip External Dependencies (Offline Mode)
# Docusaurus injects Google Fonts and Algolia Search by default. We strip them from 
# the compiled HTML so the browser doesn't hang in an air-gapped environment.
RUN find build -type f -name "*.html" -exec sed -i -E 's/<link[^>]*fonts.(googleapis|gstatic).com[^>]*>//g' {} + || true && \
    find build -type f -name "*.html" -exec sed -i -E 's/<script[^>]*google-analytics[^>]*>.*<\/script>//g' {} + || true

# --- Stage 2: Serve the Offline Site ---
FROM nginx:alpine

# Docusaurus outputs the compiled static site to the 'build' directory
COPY --from=builder /app/build /usr/share/nginx/html
COPY default.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
