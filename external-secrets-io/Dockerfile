# --- Stage 1: Build the Static Site ---
FROM python:3.12-slim AS builder

WORKDIR /app

RUN apt-get update && apt-get install -y git && rm -rf /var/lib/apt/lists/*
RUN git config --global --add safe.directory '*'

# Copy the external-secrets repository source (docs live in docs/, mkdocs config
# lives in hack/api-docs/mkdocs.yml and points back at ../../docs, ../../overrides)
COPY external-secrets-src/ .

RUN pip install --no-cache-dir -r hack/api-docs/requirements.txt

# --- OFFLINE PATCHING (pre-build) ---
# 1. Disable Material's Google Fonts auto-loading (falls back to system fonts,
#    otherwise the theme injects <link> tags pointing at fonts.googleapis.com).
RUN sed -i '/^theme:/a\  font: false' hack/api-docs/mkdocs.yml

# 2. Strip the Google Analytics config block so mkdocs-material never injects
#    the googletagmanager.com gtag.js loader.
RUN sed -i '/analytics:/,+2d' hack/api-docs/mkdocs.yml

# 3. Remove the Scarf.sh tracking pixel baked into the custom footer override.
RUN sed -i '/static\.scarf\.sh/d' overrides/main.html

# 4. Strip repo_url/repo_name/edit_uri so mkdocs-material never renders the
#    header "source facts" widget (star/fork/release counts), whose JS makes
#    runtime fetch() calls to api.github.com.
RUN sed -i -E '/^(repo_url|repo_name|edit_uri):/d' hack/api-docs/mkdocs.yml

# docs/api/spec.md (and the rest of docs/api/*.md) are committed upstream, so no
# Go toolchain / `make docs` generation step is required for a straightforward build.
RUN mkdocs build -f hack/api-docs/mkdocs.yml -d /out && test -f /out/index.html || (echo "FATAL: MkDocs generated no content" && exit 1)

# --- OFFLINE PATCHING (post-build) ---
# Belt-and-braces sweep of the rendered HTML in case any font/analytics markup
# survived the pre-build patch.
RUN find /out -type f -name "*.html" -exec sed -i -E 's#<link[^>]*fonts\.(googleapis|gstatic)\.com[^>]*>##g' {} + \
    && find /out -type f -name "*.html" -exec sed -i -E '/(googletagmanager|google-analytics)\.com/d' {} + \
    && find /out -type f -name "*.html" -exec sed -i -E 's#<img[^>]*scarf\.sh[^>]*/?>##g' {} +

RUN chmod -R 755 /out

# --- Stage 2: Serve the Offline Site ---
FROM nginx:alpine

COPY --from=builder /out /usr/share/nginx/html
COPY default.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
