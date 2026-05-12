# --- Stage 1: Build the Static Site ---
FROM golang:1.21.9-bookworm AS builder

USER root
WORKDIR /app

# 1. Install OS dependencies mirroring FluxCD's Alpine list (translated to Debian)
RUN apt-get update && apt-get install -y \
    jq \
    rsync \
    build-essential \
    libffi-dev \
    python3 \
    python3-pip \
    python3-venv \
    python3-dev \
    curl \
    && rm -rf /var/lib/apt/lists/*

# 2. Lock in Node.js 18 (required by netlify.toml)
RUN curl -fsSL https://deb.nodesource.com/setup_18.x | bash - && \
    apt-get install -y nodejs && \
    rm -rf /var/lib/apt/lists/*

# 3. Lock in Hugo Extended 0.122.0 (required by netlify.toml)
ENV HUGO_VERSION=0.122.0
RUN curl -L "https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_Linux-64bit.tar.gz" \
    | tar -xz -C /usr/local/bin hugo

RUN git config --global safe.directory '*'

# Copy the FluxCD website source
COPY fluxcd-website/ .

# 4. Set up Python Virtual Environment and install requirements
RUN python3 -m venv .venv
ENV PATH="/app/.venv/bin:$PATH"
RUN if [ -f requirements.txt ]; then pip install --no-cache-dir -r requirements.txt; fi

# 5. Install Node.js dependencies
RUN npm install

# 6. Download Hugo Modules
RUN hugo mod vendor || true

# 7. --- OFFLINE PATCHING & MOCKING ---
# Mock the missing calendar data so the homepage "Next Event" widget doesn't crash the build
RUN mkdir -p data && echo '[]' > data/calendar.json && echo '[]' > data/calendar.yaml

# Delete tracking, remove fonts, and strip SRI hashes
RUN find . -type f -name "*.html" -exec sed -i -E '/_internal\/(google_analytics|disqus)/d' {} + || true && \
    find layouts -type f -name "*.html" -exec sed -i -E 's/<link[^>]*fonts.(googleapis|gstatic).com[^>]*>//g' {} + || true && \
    find . -type f \( -name "*.html" -o -name "*.js" \) -exec sed -i -e 's/ integrity="sha[0-9]*-[^"]*"//g' -e 's/ crossorigin="anonymous"//g' {} + || true

# 8. Build the site for production
RUN hugo --minify --environment production --baseURL "/" -d /app/public

# --- Stage 2: Serve the Offline Site ---
FROM nginx:alpine

COPY --from=builder /app/public /usr/share/nginx/html
COPY default.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
