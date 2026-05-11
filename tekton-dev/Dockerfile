# --- Stage 1: Build the Static Site ---
FROM klakegg/hugo:0.107.0-ext-ubuntu AS builder

WORKDIR /src

# 1. Delete the broken Node 18 repository, then update
RUN rm -f /etc/apt/sources.list.d/nodesource.list && \
    apt-get update && \
    apt-get install -y git build-essential python3-venv python3-pip curl && \
    rm -rf /var/lib/apt/lists/*

# 2. Lock in your strict Node v22.14.0 and NPM v10.9.2 requirements
RUN curl -fsSL https://deb.nodesource.com/setup_22.x | bash - && \
    apt-get install -y nodejs && \
    npm install -g npm@10.9.2

RUN git config --global --add safe.directory '*'

COPY tekton-website/ .

# 3. Install Node modules using `npm install`
RUN npm install

# 4. Execute the Tekton Sync Script
RUN python3 -m venv .venv && \
    . .venv/bin/activate && \
    pip3 install -r requirements.txt && \
    ./sync/sync.py

# 5. --- OFFLINE ROUTING & CDN PATCH ---
# Strip hardcoded domains, remove SRI hashes, and delete external font dependencies
RUN find . \( -path "./.git" -o -path "./node_modules" -o -path "./.venv" -o -path "./public" \) -prune -o \
      -type f \( -name "*.html" -o -name "*.js" -o -name "*.toml" -o -name "*.yaml" \) \
      -exec sed -i -e 's|https://tekton\.dev||g' -e 's/ integrity="sha[0-9]*-[^"]*"//g' {} + || true && \
    find layouts -type f -name "*.html" -exec sed -i -E 's|<link[^>]*fonts.googleapis.com[^>]*>||g' {} + || true && \
    find layouts -type f -name "*.html" -exec sed -i -E 's|<link[^>]*fonts.gstatic.com[^>]*>||g' {} + || true

# 6. Build the site for production using root-relative paths
RUN hugo --minify --environment production --baseURL "/" -d /src/public

# --- Stage 2: Serve the Offline Site ---
FROM nginx:alpine

COPY --from=builder /src/public /usr/share/nginx/html
COPY default.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]