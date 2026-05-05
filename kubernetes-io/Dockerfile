# --- Stage 1: Build the Static Sites ---
FROM hugomods/hugo:exts-0.125.4 AS builder

USER root
WORKDIR /app
RUN git config --global safe.directory '*'
COPY k8s-website/ .

ARG OLD_VERSIONS=""

# 1. Build the older versions into subdirectories using relative URLs
RUN for version in $OLD_VERSIONS; do \
      echo "Building $version..."; \
      git reset --hard; \
      git checkout $version; \
      sed -i 's/(eq .Section "case-studies")/false/g' layouts/partials/css.html || true; \
      npm ci; \
      V_DIR=$(echo $version | sed 's/release-/v/'); \
      HUGO_RELATIVEURLS=true hugo --minify --environment production --renderSegments en --baseURL "/$V_DIR/" -d /app/public/$V_DIR; \
    done

# 2. Build the main/current version into the root directory
RUN echo "Building main version..." && \
    git reset --hard && \
    git checkout main && \
    sed -i 's/(eq .Section "case-studies")/false/g' layouts/partials/css.html || true && \
    npm ci && \
    HUGO_RELATIVEURLS=true hugo --minify --environment production --renderSegments en --baseURL "/" -d /app/public

# 3. Patch any remaining hardcoded domain names across all generated HTML, JS, and JSON files.
# Covers the version dropdown menu and any search index / bundled JS that references kubernetes.io.
RUN find /app/public -type f \( -name "*.html" -o -name "*.js" -o -name "*.json" \) -exec sed -i \
    -e 's|https://v1-\([0-9]\{2\}\)\.docs\.kubernetes\.io|/v1.\1|g' \
    -e 's|https://kubernetes\.io/v1\.[0-9]\{2\}||g' \
    -e 's|https://kubernetes\.io/|/|g' \
    -e 's|https://kubernetes\.io|/|g' {} +

# --- Stage 2: Serve the Offline Site ---
FROM registry.access.redhat.com/hi/nginx:latest

COPY --from=builder /app/public /usr/share/nginx/html

COPY default.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
