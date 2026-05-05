# --- Stage 1: Build the Static Sites ---
FROM hugomods/hugo:exts-0.125.4 AS builder

USER root
WORKDIR /app

# Prevent Git security warnings when running as root in the container
RUN git config --global safe.directory '*'

COPY k8s-website/ .

# Define which older versions to build (Space separated)
ARG OLD_VERSIONS=""

# 1. Build the older versions into subdirectories
RUN for version in $OLD_VERSIONS; do \
      echo "Building $version..."; \
      git reset --hard; \
      git checkout $version; \
      # Try to patch the CSS file, but don't fail if older versions don't have this exact line
      sed -i 's/(eq .Section "case-studies")/false/g' layouts/partials/css.html || true; \
      npm ci; \
      # Extract the "1.29" from "release-1.29" to use as the folder name
      V_DIR=$(echo $version | sed 's/release-/v/'); \
      hugo --minify --environment production --renderSegments en --baseURL "http://REPLACE_ME_BASE_URL/$V_DIR/" -d /app/public/$V_DIR; \
    done

# 2. Build the main/current version into the root directory
RUN echo "Building main version..." && \
    git reset --hard && \
    git checkout main && \
    sed -i 's/(eq .Section "case-studies")/false/g' layouts/partials/css.html || true && \
    npm ci && \
    hugo --minify --environment production --renderSegments en --baseURL "http://REPLACE_ME_BASE_URL/" -d /app/public

# --- Stage 2: Serve the Offline Site ---
FROM nginx:alpine

COPY --from=builder /app/public /usr/share/nginx/html

# Update the startup script to patch the version dropdown links AND the baseURL
RUN echo '#!/bin/sh' > /docker-entrypoint.d/99-set-baseurl.sh && \
    # First, intercept the dropdown menu URLs (https://v1-XX.docs.kubernetes.io) and reroute them to our dummy URL paths
    echo 'find /usr/share/nginx/html -type f -name "*.html" -exec sed -i "s|https://v1-\([0-9]\{2\}\)\.docs\.kubernetes\.io/|http://REPLACE_ME_BASE_URL/v1.\1/|g" {} +' >> /docker-entrypoint.d/99-set-baseurl.sh && \
    # Second, replace the dummy URL with the actual runtime $BASE_URL
    echo 'if [ -n "$BASE_URL" ]; then' >> /docker-entrypoint.d/99-set-baseurl.sh && \
    echo '  echo "Dynamically updating baseURL to $BASE_URL..."' >> /docker-entrypoint.d/99-set-baseurl.sh && \
    echo '  find /usr/share/nginx/html -type f \( -name "*.html" -o -name "*.xml" -o -name "*.css" -o -name "*.js" \) -exec sed -i "s|http://REPLACE_ME_BASE_URL/|$BASE_URL|g" {} +' >> /docker-entrypoint.d/99-set-baseurl.sh && \
    echo 'fi' >> /docker-entrypoint.d/99-set-baseurl.sh && \
    chmod +x /docker-entrypoint.d/99-set-baseurl.sh

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]