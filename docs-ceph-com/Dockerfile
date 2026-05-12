# --- Stage 1: Build the Static Site ---
FROM python:3.11-slim AS builder

WORKDIR /app

# 1. Install OS dependencies
RUN apt-get update && apt-get install -y \
    git \
    ditaa \
    graphviz \
    doxygen \
    ant \
    default-jdk \
    build-essential \
    libxml2-dev \
    libxslt1-dev \
    cython3 \
    && rm -rf /var/lib/apt/lists/*

RUN git config --global --add safe.directory '*'

COPY ceph-repo/ .

# Define the historical versions natively inside the Dockerfile
ENV OLD_VERSIONS="tentacle squid reef quincy"

# Pre-create chunk directories for 4 older versions + 1 main version
RUN mkdir -p /out/chunk_main /out/chunk_old_1 /out/chunk_old_2 /out/chunk_old_3 /out/chunk_old_4

# 2. Build historical versions
RUN count=1 && for version in $OLD_VERSIONS; do \
      echo "Building $version..."; \
      # Ceph branches occasionally require cleaning between checkouts
      git reset --hard && git clean -fdx && git checkout $version || true; \
      \
      # Use an isolated virtual environment per branch to prevent dependency clashes
      python3 -m venv .venv_$version; \
      export PATH="/app/.venv_$version/bin:$PATH"; \
      pip install --no-cache-dir wheel; \
      pip install --no-cache-dir -r admin/doc-requirements.txt -r admin/doc-python-common-requirements.txt || true; \
      BUILD_DOC=1 pip install --no-cache-dir -r admin/doc-pybind.txt || true; \
      \
      # Build Sphinx to a specific subfolder matching the branch name (e.g. /squid/)
      sphinx-build -a -b dirhtml -d doctrees_$version doc/ /out/chunk_old_${count}/$version || true; \
      \
      # Build JavaDocs safely
      if [ -f src/java/build.xml ]; then \
        ant -buildfile src/java/build.xml docs || true; \
        mkdir -p /out/chunk_old_${count}/$version/cephfs/api/libcephfs-java/javadoc; \
        cp -a src/java/doc/* /out/chunk_old_${count}/$version/cephfs/api/libcephfs-java/javadoc/ || true; \
      fi; \
      \
      # Strip external fonts for offline capability
      find /out/chunk_old_${count}/$version -type f -name "*.html" -exec sed -i -E 's/<link[^>]*fonts.(googleapis|gstatic).com[^>]*>//g' {} + || true; \
      count=$((count + 1)); \
    done

# 3. Build the "latest" (main) version
RUN echo "Building main version (latest)..." && \
    git reset --hard && git clean -fdx && git checkout main && \
    \
    python3 -m venv .venv_main && \
    export PATH="/app/.venv_main/bin:$PATH" && \
    pip install --no-cache-dir wheel && \
    pip install --no-cache-dir -r admin/doc-requirements.txt -r admin/doc-python-common-requirements.txt && \
    BUILD_DOC=1 pip install --no-cache-dir -r admin/doc-pybind.txt && \
    \
    # Build main directly into the chunk root
    sphinx-build -a -b dirhtml -d doctrees_main doc/ /out/chunk_main && \
    \
    ant -buildfile src/java/build.xml docs || true && \
    mkdir -p /out/chunk_main/cephfs/api/libcephfs-java/javadoc && \
    cp -a src/java/doc/* /out/chunk_main/cephfs/api/libcephfs-java/javadoc/ || true && \
    \
    find /out/chunk_main -type f -name "*.html" -exec sed -i -E 's/<link[^>]*fonts.(googleapis|gstatic).com[^>]*>//g' {} + || true

# --- Stage 2: Serve the Offline Site ---
FROM nginx:alpine

# Merge all chunks into Nginx
COPY --from=builder /out/chunk_old_1 /usr/share/nginx/html
COPY --from=builder /out/chunk_old_2 /usr/share/nginx/html
COPY --from=builder /out/chunk_old_3 /usr/share/nginx/html
COPY --from=builder /out/chunk_old_4 /usr/share/nginx/html
COPY --from=builder /out/chunk_main /usr/share/nginx/html

COPY default.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
