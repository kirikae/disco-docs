FROM python:3.12-slim AS builder

WORKDIR /app

RUN apt-get update && apt-get install -y git && rm -rf /var/lib/apt/lists/*
RUN git config --global --add safe.directory '*'

COPY knative-docs/ .

RUN pip install --no-cache-dir -r requirements.txt

# --- OFFLINE PATCHING (pre-build) ---
# Drop the Google Analytics config so Material doesn't emit the
# googletagmanager.com loader script, and strip the CDN-hosted jquery/
# bootstrap/cookie-consent scripts (the site's cookie-consent banner exists
# solely to ask for GA tracking consent, which is moot offline).
RUN sed -i '/^  analytics:/,+2d' mkdocs.yml && \
    sed -i '/code\.jquery\.com\|stackpath\.bootstrapcdn\.com\|cookie-consent\.js/d' overrides/main.html

# Single-version build: only the tip of the default (main) branch is built.
# See knative-dev/README notes in the PR description for why multi-version
# (upstream's hack/build.sh) was not replicated here.
RUN mkdocs build -f mkdocs.yml -d /out && \
    test -f /out/index.html || (echo "FATAL: MkDocs generated no content!" && exit 1)

# --- OFFLINE PATCHING (post-build) ---
# Strip Google Fonts <link> tags from the generated HTML.
RUN find /out -type f -name "*.html" -exec sed -i -E 's#<link[^>]*fonts\.(googleapis|gstatic)\.com[^>]*>##g' {} +

RUN chmod -R 755 /out

FROM nginx:alpine

COPY --from=builder /out /usr/share/nginx/html

COPY default.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
