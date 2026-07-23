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
      for jq in $(grep -rhoE 'https://code\.jquery\.com/jquery-[0-9.]+\.min\.js' layouts | sort -u); do wget -qO "static/js/$(basename "$jq")" "$jq" || true; done; \
      find layouts -type f -name "*.html" -exec sed -i -E 's#https://code\.jquery\.com/(jquery-[0-9.]+\.min\.js)#/js/\1#g' {} + || true; \
      printf '\n$web-font-path: "/vendor/fonts/open-sans.css";\n' >> assets/scss/_variables_project_after_bs.scss || true; \
      sed -i -E 's#<script src="https://cdn\.jsdelivr\.net/npm/mermaid@[^"]*"[^>]*></script>#<script src="/vendor/mermaid/mermaid.min.js"></script>#' layouts/partials/scripts/mermaid.html || true; \
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
    for jq in $(grep -rhoE 'https://code\.jquery\.com/jquery-[0-9.]+\.min\.js' layouts | sort -u); do wget -qO "static/js/$(basename "$jq")" "$jq" || true; done && \
    find layouts -type f -name "*.html" -exec sed -i -E 's#https://code\.jquery\.com/(jquery-[0-9.]+\.min\.js)#/js/\1#g' {} + || true && \
    printf '\n$web-font-path: "/vendor/fonts/open-sans.css";\n' >> assets/scss/_variables_project_after_bs.scss && \
    sed -i -E 's#<script src="https://cdn\.jsdelivr\.net/npm/mermaid@[^"]*"[^>]*></script>#<script src="/vendor/mermaid/mermaid.min.js"></script>#' layouts/partials/scripts/mermaid.html && \
    find . \( -path "./.git" -o -path "./node_modules" -o -path "./public" \) -prune -o \
      -type f \( -name "*.html" -o -name "*.js" -o -name "*.json" -o -name "*.toml" -o -name "*.yaml" \) \
      -exec sed -i -e 's|https://v1-\([0-9]\{2\}\)\.docs\.kubernetes\.io|/v1.\1|g' -e 's|https://kubernetes\.io/|/main-version/|g' -e 's|https://kubernetes\.io|/main-version|g' {} + || true && \
    hugo --minify --environment production --baseURL "/" -d /app/public

# Vendor Mermaid and the Open Sans web font so pages render fully offline.
# Both live once at the site root under /vendor/ and are referenced by absolute
# URLs, so every built version shares a single copy (the source patches above
# repoint the site at these paths):
#   - Mermaid: the diagram partial injects a <script> that defaults to a jsDelivr
#     CDN; its src was patched to /vendor/mermaid/mermaid.min.js. v9.3.0 is a
#     single self-contained UMD bundle, matching the version the site references.
#   - Fonts: Docsy compiles `@import url($web-font-path)` into its stylesheet,
#     defaulting to Google Fonts, and (when $td-enable-google-fonts is on) also
#     prepends "Open Sans" to the body font stack. We keep google-fonts enabled
#     (so the stack still names Open Sans) but repoint $web-font-path to
#     /vendor/fonts/open-sans.css. The override goes in _variables_project_after_bs.scss
#     because Docsy sets $web-font-path without `!default` (an earlier override
#     would be clobbered); that file is imported after _variables but before the
#     `@import`. We fetch the exact Open Sans CSS Docsy asks for, mirror the woff2
#     files it lists, and rewrite the URLs to local paths.
# This directory (/app/public/vendor) is not a v1.* version dir, so the chunking
# step below sweeps it into chunk_main and it ends up served at /vendor/.
RUN set -eu; \
    npm install --no-save --prefix /opt/mermaid mermaid@9.3.0; \
    mkdir -p /app/public/vendor/mermaid; \
    cp /opt/mermaid/node_modules/mermaid/dist/mermaid.min.js /app/public/vendor/mermaid/; \
    UA="Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36"; \
    mkdir -p /app/public/vendor/fonts/files; \
    wget -q -U "$UA" -O /tmp/open-sans.css "https://fonts.googleapis.com/css?family=Open+Sans:300,300i,400,400i,700,700i&display=swap"; \
    for url in $(grep -oE 'https://fonts\.gstatic\.com[^)]+\.woff2' /tmp/open-sans.css | sort -u); do \
      wget -q -O "/app/public/vendor/fonts/files/$(basename "$url")" "$url"; \
    done; \
    sed -E 's#https://fonts\.gstatic\.com[^)]+/([^/)]+\.woff2)#files/\1#g' /tmp/open-sans.css > /app/public/vendor/fonts/open-sans.css

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
