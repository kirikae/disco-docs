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

# 3. Patch any remaining hardcoded domain names
RUN find /app/public -type f \( -name "*.html" -o -name "*.js" -o -name "*.json" \) -exec sed -i \
    -e 's|https://v1-\([0-9]\{2\}\)\.docs\.kubernetes\.io|/v1.\1|g' \
    -e 's|https://kubernetes\.io/v1\.[0-9]\{2\}||g' \
    -e 's|https://kubernetes\.io/|/|g' \
    -e 's|https://kubernetes\.io|/|g' {} +

# 4. CHUNKING LOGIC: 1 Version Per Chunk
# We pre-create 5 buckets for historical versions, matching the Makefile's `tail -n 5`.
RUN mkdir -p /out/chunk_1 /out/chunk_2 /out/chunk_3 \
             /out/chunk_4 /out/chunk_5 /out/chunk_main && \
    cd /app/public && \
    dirs=$(find . -maxdepth 1 -type d -name "v1.*" | sed 's|^\./||') && \
    count=1 && \
    for d in $dirs; do \
      # If we somehow build more than 5 versions, dump the overflow into bucket 5
      bucket=$count; \
      if [ $bucket -gt 5 ]; then bucket=5; fi; \
      mv "$d" "/out/chunk_${bucket}/"; \
      count=$((count + 1)); \
    done && \
    # Move whatever is left (the 'main' branch files) into the final chunk
    find . -mindepth 1 -maxdepth 1 -exec mv {} /out/chunk_main/ \;

# --- Stage 2: Serve the Offline Site ---
FROM nginx:alpine

# Copy each version layer sequentially. 
# Empty source directories safely create 0-byte layers.
COPY --from=builder /out/chunk_1 /usr/share/nginx/html
COPY --from=builder /out/chunk_2 /usr/share/nginx/html
COPY --from=builder /out/chunk_3 /usr/share/nginx/html
COPY --from=builder /out/chunk_4 /usr/share/nginx/html
COPY --from=builder /out/chunk_5 /usr/share/nginx/html
COPY --from=builder /out/chunk_main /usr/share/nginx/html

COPY default.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
