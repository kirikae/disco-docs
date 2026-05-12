# --- Stage 1: Build the Static Site ---
FROM python:3.11-slim AS builder

WORKDIR /app

# 1. Install Sphinx dependencies (Ceph explicitly requires ditaa and graphviz for diagrams)
# We also include build-essential as some Python modules require compiling C headers.
RUN apt-get update && apt-get install -y \
    git \
    ditaa \
    graphviz \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

RUN git config --global --add safe.directory '*'

# Copy the Ceph repository
COPY ceph-repo/ .

# 2. Install Ceph's explicit documentation requirements
# We extract these directly from how Ceph configures their native ReadTheDocs environment
RUN pip install --no-cache-dir \
    -r admin/doc-requirements.txt \
    -r admin/doc-read-the-docs.txt \
    -r admin/doc-pybind.txt

# 3. Build the Sphinx documentation
# The -b dirhtml flag is crucial: it forces Sphinx to output directories with index.html 
# files inside them (e.g. /docs/index.html) instead of standard /docs.html files!
RUN sphinx-build -b dirhtml doc/ /app/site/

# 4. Strip External Dependencies (Offline Mode)
# Remove Google Fonts from the Sphinx RTD theme to prevent browser network timeouts
RUN find /app/site -type f -name "*.html" -exec sed -i -E 's|<link[^>]*fonts.(googleapis|gstatic).com[^>]*>||g' {} + || true

# --- Stage 2: Serve the Offline Site ---
FROM nginx:alpine

COPY --from=builder /app/site /usr/share/nginx/html
COPY default.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
