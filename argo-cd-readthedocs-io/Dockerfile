# --- Stage 1: Build the Static Site ---
FROM python:3.12-slim AS builder

WORKDIR /app

RUN apt-get update && apt-get install -y git jq && rm -rf /var/lib/apt/lists/*
RUN git config --global --add safe.directory '*'

COPY argo-cd-repo/ .

ARG OLD_VERSIONS=""

# Pre-create 5 chunk directories for historical versions, plus main
RUN mkdir -p /out/chunk_main /out/chunk_old_1 /out/chunk_old_2 /out/chunk_old_3 /out/chunk_old_4 /out/chunk_old_5

# Initialize the versions.json file with the main branch (latest)
RUN echo "[]" > /tmp/versions.json && \
    jq '.[0] = {"version": "latest", "title": "latest (main)", "aliases": []}' /tmp/versions.json > /tmp/v2.json && mv /tmp/v2.json /tmp/versions.json

# 1. Build the 5 historical/stable versions
RUN count=1 && for version in $OLD_VERSIONS; do \
      echo "Building $version..."; \
      git reset --hard && git clean -fdx && git checkout $version || true; \
      \
      pip install --no-cache-dir -r docs/requirements.txt || true; \
      \
      # Delete site_url to bypass strict URI validation
      sed -i 's/^site_url:.*//g' mkdocs.yml || true; \
      \
      V_DIR=$(echo $version | sed 's/release-/v/'); \
      \
      # If this is the FIRST iteration (highest version), we treat it as 'stable'
      if [ "$count" -eq 1 ]; then \
          echo "Tagging $V_DIR as STABLE..."; \
          mkdocs build -d /out/chunk_old_${count}/stable || exit 1; \
          # Duplicate stable build into its actual version directory
          cp -r /out/chunk_old_${count}/stable /out/chunk_old_${count}/$V_DIR; \
          \
          jq ". += [{\"version\": \"stable\", \"title\": \"stable ($V_DIR)\", \"aliases\": [\"stable\"]}]" /tmp/versions.json > /tmp/v2.json && mv /tmp/v2.json /tmp/versions.json; \
          jq ". += [{\"version\": \"$V_DIR\", \"title\": \"$V_DIR\", \"aliases\": []}]" /tmp/versions.json > /tmp/v2.json && mv /tmp/v2.json /tmp/versions.json; \
      else \
          mkdocs build -d /out/chunk_old_${count}/$V_DIR || exit 1; \
          jq ". += [{\"version\": \"$V_DIR\", \"title\": \"$V_DIR\", \"aliases\": []}]" /tmp/versions.json > /tmp/v2.json && mv /tmp/v2.json /tmp/versions.json; \
      fi; \
      \
      # Strip external fonts
      find /out/chunk_old_${count}/ -type f -name "*.html" -exec sed -i -E 's/<link[^>]*fonts.(googleapis|gstatic).com[^>]*>//g' {} + || true; \
      count=$((count + 1)); \
    done

# 2. Build the main version
RUN echo "Building main version (latest)..." && \
    git reset --hard && git clean -fdx && git checkout master && \
    pip install --no-cache-dir -r docs/requirements.txt && \
    \
    sed -i 's/^site_url:.*//g' mkdocs.yml || true && \
    \
    # Execute MkDocs and FORCE a failure if it fails to build
    mkdocs build -d /out/chunk_main || exit 1 && \
    \
    # NUCLEAR CHECK: Abort the Docker build if the homepage is missing
    test -f /out/chunk_main/index.html || (echo "FATAL: MkDocs generated no content for main!" && exit 1) && \
    \
    find /out/chunk_main -type f -name "*.html" -exec sed -i -E 's/<link[^>]*fonts.(googleapis|gstatic).com[^>]*>//g' {} + || true && \
    cp /tmp/versions.json /out/chunk_main/versions.json

# 3. Fix file permissions to guarantee Nginx can read the files
RUN chmod -R 755 /out

# --- Stage 2: Serve the Offline Site ---
FROM nginx:alpine

# Forcefully wipe the Nginx default files so you never see the welcome page again
RUN rm -rf /usr/share/nginx/html/*

# Merge all chunks into Nginx
COPY --from=builder /out/chunk_old_1/ /usr/share/nginx/html/
COPY --from=builder /out/chunk_old_2/ /usr/share/nginx/html/
COPY --from=builder /out/chunk_old_3/ /usr/share/nginx/html/
COPY --from=builder /out/chunk_old_4/ /usr/share/nginx/html/
COPY --from=builder /out/chunk_old_5/ /usr/share/nginx/html/
COPY --from=builder /out/chunk_main/ /usr/share/nginx/html/

COPY default.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
