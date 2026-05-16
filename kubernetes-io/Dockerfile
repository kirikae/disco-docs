FROM hugomods/hugo:exts-0.144.2 AS builder

USER root
WORKDIR /app
RUN git config --global safe.directory '*'
COPY k8s-website/ .

ARG OLD_VERSIONS=""

RUN wget -qO- https://github.com/gohugoio/hugo/releases/download/v0.125.4/hugo_extended_0.125.4_linux-amd64.tar.gz | tar -xz -C /usr/local/bin hugo && mv /usr/local/bin/hugo /usr/local/bin/hugo-legacy

RUN for version in $OLD_VERSIONS; do \
      git fetch --depth 1 https://github.com/kubernetes/website.git $version:$version || true; \
      git reset --hard && git clean -fdx; \
      git checkout $version || true; \
      find content -mindepth 1 -maxdepth 1 -type d ! -name 'en' -exec rm -rf {} + || true; \
      sed -i 's/(eq .Section "case-studies")/false/g' layouts/partials/css.html || true; \
      npm ci; \
      hugo mod vendor || true; \
      mkdir -p static/js; \
      wget -qO static/js/jquery-3.6.0.min.js "https://code.jquery.com/jquery-3.6.0.min.js" || true; \
      find layouts -type f -name "*.html" -exec sed -i 's|https://code.jquery.com/jquery-3.6.0.min.js|/js/jquery-3.6.0.min.js|g' {} + || true; \
      find layouts -type f -name "*.html" -exec sed -i -E 's|<link[^>]*fonts.googleapis.com[^>]*>||g' {} + || true; \
      find layouts -type f -name "*.html" -exec sed -i -E 's|<link[^>]*fonts.gstatic.com[^>]*>||g' {} + || true; \
      find . \( -path "./.git" -o -path "./node_modules" -o -path "./public" \) -prune -o \
        -type f \( -name "*.html" -o -name "*.js" -o -name "*.json" -o -name "*.toml" -o -name "*.yaml" \) \
        -exec sed -i -e 's|https://v1-\([0-9]\{2\}\)\.docs\.kubernetes\.io|/v1.\1|g' -e 's|https://kubernetes\.io/|/main-version/|g' -e 's|https://kubernetes\.io|/main-version|g' {} + || true; \
      V_DIR=$(echo $version | sed 's/release-/v/'); \
      if ! hugo-legacy --minify --environment production --baseURL "/$V_DIR/" -d /app/public/$V_DIR; then \
        hugo --minify --environment production --baseURL "/$V_DIR/" -d /app/public/$V_DIR || exit 1; \
      fi; \
    done

RUN git reset --hard && git clean -fdx && \
    git checkout main && \
    find content -mindepth 1 -maxdepth 1 -type d ! -name 'en' -exec rm -rf {} + || true && \
    sed -i 's/(eq .Section "case-studies")/false/g' layouts/partials/css.html || true && \
    npm ci && \
    hugo mod vendor || true && \
    mkdir -p static/js && \
    wget -qO static/js/jquery-3.6.0.min.js "https://code.jquery.com/jquery-3.6.0.min.js" || true && \
    find layouts -type f -name "*.html" -exec sed -i 's|https://code.jquery.com/jquery-3.6.0.min.js|/js/jquery-3.6.0.min.js|g' {} + || true && \
    find layouts -type f -name "*.html" -exec sed -i -E 's|<link[^>]*fonts.googleapis.com[^>]*>||g' {} + || true && \
    find layouts -type f -name "*.html" -exec sed -i -E 's|<link[^>]*fonts.gstatic.com[^>]*>||g' {} + || true && \
    find . \( -path "./.git" -o -path "./node_modules" -o -path "./public" \) -prune -o \
      -type f \( -name "*.html" -o -name "*.js" -o -name "*.json" -o -name "*.toml" -o -name "*.yaml" \) \
      -exec sed -i -e 's|https://v1-\([0-9]\{2\}\)\.docs\.kubernetes\.io|/v1.\1|g' -e 's|https://kubernetes\.io/|/main-version/|g' -e 's|https://kubernetes\.io|/main-version|g' {} + || true && \
    hugo --minify --environment production --baseURL "/" -d /app/public

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
