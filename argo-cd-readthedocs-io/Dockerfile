# --- Stage 1: Build the Static Site ---
FROM python:3.12-slim AS builder

WORKDIR /app

# Install jq to easily construct our JSON array, and git for branch checkouts
RUN apt-get update && apt-get install -y git jq && rm -rf /var/lib/apt/lists/*
RUN git config --global --add safe.directory '*'

COPY argo-cd-repo/ .

ARG OLD_VERSIONS=""

# Pre-create our chunk directories to keep Docker layers small
RUN mkdir -p /out/chunk_main /out/chunk_old_1 /out/chunk_old_2 /out/chunk_old_3

# Initialize the versions.json file with the main branch
RUN echo "[]" > /tmp/versions.json && \
    jq '.[0] = {"version": "latest", "title": "latest (main)", "aliases": []}' /tmp/versions.json > /tmp/v2.json && mv /tmp/v2.json /tmp/versions.json

# 1. Build historical versions
RUN count=1 && for version in $OLD_VERSIONS; do \
      echo "Building $version..."; \
      git reset --hard && git checkout $version; \
      \
      # Install dependencies (they sometimes change between releases)
      pip install --no-cache-dir -r docs/requirements.txt || true; \
      \
      # Patch mkdocs.yml to use an EMPTY STRING for native MkDocs offline mode
      sed -i "s|^site_url:.*|site_url: ''|g" mkdocs.yml || true; \
      # Activate the native MkDocs Material version dropdown via inline Python
      python3 -c "import yaml; d=yaml.safe_load(open('mkdocs.yml')); d.setdefault('extra',{})['version']={'provider':'mike'}; yaml.dump(d, open('mkdocs.yml','w'))" || true; \
      \
      V_DIR=$(echo $version | sed 's/release-/v/'); \
      \
      # Add this version to the JSON dropdown menu array
      jq ". += [{\"version\": \"$V_DIR\", \"title\": \"$V_DIR\", \"aliases\": []}]" /tmp/versions.json > /tmp/v2.json && mv /tmp/v2.json /tmp/versions.json; \
      \
      # Build this old version directly into its chunk directory
      mkdocs build -d /out/chunk_old_${count}/$V_DIR; \
      # Strip external fonts
      find /out/chunk_old_${count}/$V_DIR -type f -name "*.html" -exec sed -i -E 's|<link[^>]*fonts.(googleapis|gstatic).com[^>]*>||g' {} + || true; \
      count=$((count + 1)); \
    done

# 2. Build the main version
RUN echo "Building main version..." && \
    git reset --hard && git checkout master && \
    pip install --no-cache-dir -r docs/requirements.txt && \
    # Patch mkdocs.yml to use an EMPTY STRING for native MkDocs offline mode
    sed -i "s|^site_url:.*|site_url: ''|g" mkdocs.yml || true && \
    python3 -c "import yaml; d=yaml.safe_load(open('mkdocs.yml')); d.setdefault('extra',{})['version']={'provider':'mike'}; yaml.dump(d, open('mkdocs.yml','w'))" || true && \
    mkdocs build -d /out/chunk_main && \
    find /out/chunk_main -type f -name "*.html" -exec sed -i -E 's|<link[^>]*fonts.(googleapis|gstatic).com[^>]*>||g' {} + || true && \
    # Move the completed dropdown configuration into the main root
    cp /tmp/versions.json /out/chunk_main/versions.json

# --- Stage 2: Serve the Offline Site ---
FROM nginx:alpine

# 1. Forcefully delete the default Nginx welcome page and assets
RUN rm -rf /usr/share/nginx/html/*

# 2. Merge all chunks into Nginx
COPY --from=builder /out/chunk_old_1/ /usr/share/nginx/html/
COPY --from=builder /out/chunk_old_2/ /usr/share/nginx/html/
COPY --from=builder /out/chunk_old_3/ /usr/share/nginx/html/
COPY --from=builder /out/chunk_main/ /usr/share/nginx/html/

COPY default.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]