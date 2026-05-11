# --- Stage 1: Build the Static Site ---
FROM hugomods/hugo:exts-0.125.4 AS builder

USER root
WORKDIR /app

# Install Python 3 (Strictly required for Tekton's sync script)
RUN apt-get update && apt-get install -y python3 python3-pip python3-venv && rm -rf /var/lib/apt/lists/*

RUN git config --global safe.directory '*'
COPY tekton-website/ .

# Execute the Tekton Sync Script
# This reads config.yaml and pulls the markdown files from all the external Tekton sub-projects
RUN python3 -m venv .venv && \
    . .venv/bin/activate && \
    pip install -r requirements.txt && \
    ./sync/sync.py

# Install Node.js dependencies
RUN npm ci

# Build the site for production using relative root paths
RUN hugo --minify --environment production --baseURL "/" -d /app/public

# --- Stage 2: Serve the Offline Site ---
FROM nginx:alpine

COPY --from=builder /app/public /usr/share/nginx/html
COPY default.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]