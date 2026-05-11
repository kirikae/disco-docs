# --- Stage 1: Build the Static Site ---
# Use the exact base image from Tekton's docker-compose
FROM klakegg/hugo:0.107.0-ext-ubuntu AS builder

# The Tekton Dockerfile sets up in /src
WORKDIR /src

# 1. Install the base OS prerequisites they require
RUN apt-get update && \
    apt-get install -y git build-essential python3-venv python3-pip curl && \
    rm -rf /var/lib/apt/lists/*

# 2. Lock in your strict Node v22.14.0 and NPM v10.9.2 requirements
# The klakegg image has an older version of Node, so we overwrite it using NodeSource
RUN curl -fsSL https://deb.nodesource.com/setup_22.x | bash - && \
    apt-get install -y nodejs && \
    npm install -g npm@10.9.2

# Prevent Git security warnings
RUN git config --global --add safe.directory '*'

# Copy the Tekton website source code
COPY tekton-website/ .

# 3. Install Node modules using `npm install`
RUN npm install

# 4. Execute the Tekton Sync Script
# This downloads all the markdown files from the external Tekton sub-projects
RUN python3 -m venv .venv && \
    . .venv/bin/activate && \
    pip3 install -r requirements.txt && \
    ./sync/sync.py

# 5. Build the site for production using root-relative paths
RUN hugo --minify --environment production --baseURL "/" -d /src/public

# --- Stage 2: Serve the Offline Site ---
FROM nginx:alpine

COPY --from=builder /src/public /usr/share/nginx/html
COPY default.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]