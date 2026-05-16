# --- Stage 1: Build the Static Sites ---
FROM hugomods/hugo:exts-0.144.2 AS builder

USER root
WORKDIR /app
RUN git config --global safe.directory '*'
COPY k8s-website/ .

ARG OLD_VERSIONS=""

# 1. Build the older versions into subdirectories
RUN for version in $OLD_VERSIONS; do \
      echo "Building $version..."; \
      git reset --hard; \
      git checkout $version; \
      sed -i 's/(eq .Section "case-studies")/false/g' layouts/partials/css.html || true; \
      npm ci; \
      hugo mod vendor || true; \
      \
      # --- OFFLINE VENDORING FIX --- \
      # Download jQuery locally to the static folder \
      mkdir -p static/js; \
      curl -sSL "https://code.jquery.com/jquery-3.6.0.min.js" -o static/js/jquery-3.6.0.min.js || true; \
      # Redirect the HTML layout to use the local jQuery instead of the CDN \
      find layouts -type f -name "*.html" -exec sed -i 's|https://code.jquery.com/jquery-3.6.0.min.js|/js/jquery-3.6.0.min.js|g' {} + || true; \
      # Strip out external Google Fonts to prevent network hangs \
      find layouts -type f -name "*.html" -exec sed -i -E 's|<link[^>]*fonts.googleapis.com[^>]*>||g' {} + || true; \
      find layouts -type f -name "*.html" -exec sed -i -E 's|<link[^>]*fonts.gstatic.com[^>]*>||g' {} + || true; \
      # ----------------------------- \
      \
      # Pre-build patch for domains \
      find . \( -path "./.git" -o -path "./node_modules" -o -path "./public" \) -prune -o \
        -type f \( -name "*.html" -o -name "*.js" -o -name "*.json" -o -name "*.toml" -o -name "*.yaml" \) \
        -exec sed -i -e 's|https://v1-\([0-9]\{2\}\)\.docs\.kubernetes\.io|/v1.\1|g' -e 's|https://kubernetes\.io/|/main-version/|g' -e 's|https://kubernetes\.io|/main-version|g' {} + || true; \
      V_DIR=$(echo $version | sed 's/release-/v/'); \
      hugo --minify --environment production --renderSegments en --baseURL "/$V_DIR/" -d /app/public/$V_DIR; \
    done

# 2. Build the main/current version into the root directory
RUN echo "Building main version..." && \
    git reset --hard && \
    git checkout main && \
    sed -i 's/(eq .Section "case-studies")/false/g' layouts/partials/css.html || true && \
    npm ci && \
    hugo mod vendor || true && \
    \
    # --- OFFLINE VENDORING FIX (Main Branch) --- \
    mkdir -p static/js && \
    curl -sSL "https://code.jquery.com/jquery-3.6.0.min.js" -o static/js/jquery-3.6.0.min.js || true && \
    find layouts -type f -name "*.html" -exec sed -i 's|https://code.jquery.com/jquery-3.6.0.min.js|/js/jquery-3.6.0.min.js|g' {} + || true && \
    find layouts -type f -name "*.html" -exec sed -i -E 's|<link[^>]*fonts.googleapis.com[^>]*>||g' {} + || true && \
    find layouts -type f -name "*.html" -exec sed -i -E 's|<link[^>]*fonts.gstatic.com[^>]*>||g' {} + || true && \
    # ------------------------------------------- \
    \
    find . \( -path "./.git" -o -path "./node_modules" -o -path "./public" \) -prune -o \
      -type f \( -name "*.html" -o -name "*.js" -o -name "*.json" -o -name "*.toml" -o -name "*.yaml" \) \
      -exec sed -i -e 's|https://v1-\([0-9]\{2\}\)\.docs\.kubernetes\.io|/v1.\1|g' -e 's|https://kubernetes\.io/|/main-version/|g' -e 's|https://kubernetes\.io|/main-version|g' {} + || true && \
    hugo --minify --environment production --renderSegments en --baseURL "/" -d /app/public

# 3. CHUNKING LOGIC: 1 Version Per Chunk
RUN mkdir -p /out/chunk_old_1 /out/chunk_old_2 /out/chunk_old_3 \
             /out/chunk_old_4 /out/chunk_old_5 /out/chunk_main && \
    cd /app/public && \
    dirs=$(find . -maxdepth 1 -type d -name "v1.*" | sed 's|^\./||') && \
    count=1 && \
    for d in $dirs; do \
      bucket=$count; \
      if [ $bucket -gt 5 ]; then bucket=5; fi; \
      mv "$d" "/out/chunk_old_${bucket}/"; \
      count=$((count + 1)); \
    done && \
    find . -mindepth 1 -maxdepth 1 -exec mv {} /out/chunk_main/ \;

# --- Stage 2: Serve the Offline Site ---
FROM nginx:alpine

COPY --from=builder /out/chunk_old_1 /usr/share/nginx/html
COPY --from=builder /out/chunk_old_2 /usr/share/nginx/html
COPY --from=builder /out/chunk_old_3 /usr/share/nginx/html
COPY --from=builder /out/chunk_old_4 /usr/share/nginx/html
COPY --from=builder /out/chunk_old_5 /usr/share/nginx/html
COPY --from=builder /out/chunk_main /usr/share/nginx/html

COPY default.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
