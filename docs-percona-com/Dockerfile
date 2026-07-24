# --- Stage 1: Build the Static Site ---
#
# Percona publishes docs for many separate products (PMM, Percona Server for
# MySQL, XtraDB Cluster, the various Kubernetes Operators, Backup for
# MongoDB, ...), each usually its own MkDocs-Material repo. This build picks
# ONE of them: Percona Server for MySQL (github.com/percona/psmysql-docs,
# branch "8.4" = the current release line). PMM's own repo (percona/pmm-doc)
# is archived/unmaintained, so Percona Server for MySQL was chosen instead as
# the actively-maintained, cleanest-to-build representative of docs.percona.com.
#
# Only the single "latest" version (branch 8.4, no `mike` multi-version
# deploy) is built - the upstream repo's version history lives in separate
# git branches (8.4, 8.0, 5.7, ...) that would each need their own `mike
# deploy` pass to reproduce Percona's multi-version site; that is out of
# scope here.
FROM python:3.12-slim AS builder

WORKDIR /app

RUN apt-get update && apt-get install -y git && rm -rf /var/lib/apt/lists/*
RUN git config --global --add safe.directory '*'

COPY percona-src/ .

# mkdocs-with-pdf drags in weasyprint (and its native pango/cairo system
# libs) purely for a PDF export target this build doesn't use - the config
# below (mkdocs-base.yml) never enables a PDF plugin. Drop it to keep the
# image lean and avoid an unused, heavier build dependency.
RUN sed -i '/^mkdocs-with-pdf/d' requirements.txt
RUN pip install --no-cache-dir -r requirements.txt

# --- OFFLINE PATCHING (pre-build) ---
#
# Upstream ships two site configs:
#   - mkdocs-percona.yml: the real docs.percona.com build. It sets
#     `custom_dir: _resource/theme` and references css/version-select.css,
#     css/toctree.css, css/percona.css, css/details.css - NONE of which
#     exist in this repo. They are supplied at build time by Percona's
#     internal multi-repo doc pipeline (a shared theme/assets layer), so this
#     config is not self-buildable standalone.
#   - mkdocs-base.yml: the config mkdocs-percona.yml and the PDF config both
#     INHERIT from. It is fully self-contained (no custom_dir, no missing
#     assets) and renders the same content with the stock Material theme, so
#     it is what this build uses directly.
#
# mkdocs-base.yml still needs several offline/privacy patches:
#   1. repo_url/repo_name/edit_uri: with these set, MkDocs Material's default
#      header renders a repo-facts widget that fetches api.github.com
#      client-side on every page load. Strip them (kills the fetch and the
#      GitHub edit links, which would be dead in an offline mirror anyway).
#   2. extra.analytics: Google Analytics (G-J4J70BNH0G) - strip.
#   3. extra_css: unicons.iconscout.com and cdnjs.cloudflare.com (Font
#      Awesome) are loaded but never referenced by any class in this repo's
#      templates/content (verified via grep) - drop both external CDN lines
#      rather than vendoring unused assets. Also drop kapa.css/rating.css
#      (styling for widgets removed in patch 4/5) and osano.css (styling for
#      a cookie-consent banner script that patch 4 also removes).
#   4/5. extra_javascript: drop kapa.js (loads the "Ask Percona AI" widget
#      from widget.kapa.ai), rating.js (POSTs page-feedback to a Google Forms
#      endpoint via fetch()), consent.js (Osano cookie-consent cookie read,
#      dead without the Osano script) and version-select.js (XHR-fetches a
#      versions.json that only exists in Percona's real multi-version mike
#      deploy - not produced by this single-version build). Keep
#      promptremover.js (a local, same-page DOM helper for copy-button
#      command prompts - no network calls).
RUN python3 - <<'EOF'
import re
p = "mkdocs-base.yml"
s = open(p).read()

s = re.sub(r"^repo_name:.*\n", "", s, flags=re.M)
s = re.sub(r"^repo_url:.*\n", "", s, flags=re.M)
s = re.sub(r"^edit_uri:.*\n", "", s, flags=re.M)

for line in [
    "  - https://unicons.iconscout.com/release/v3.0.3/css/line.css\n",
    "  - https://cdnjs.cloudflare.com/ajax/libs/font-awesome/4.4.0/css/font-awesome.min.css\n",
    "  - css/osano.css\n",
    "  - css/rating.css\n",
    "  - css/kapa.css\n",
    "  - js/version-select.js\n",
    "  - js/consent.js\n",
    "  - js/rating.js\n",
    "  - js/kapa.js\n",
]:
    assert line in s, f"expected line not found, upstream layout changed: {line!r}"
    s = s.replace(line, "")

s = re.sub(r"#Google Analytics configuration \n  analytics:\n    provider: google\n    property: G-J4J70BNH0G\n", "", s)

open(p, "w").write(s)
print("patched mkdocs-base.yml OK")
EOF

# Drop the now-orphaned widget/consent source files outright, so nothing can
# load them even by accident (e.g. a stray reference from content).
RUN rm -f docs/js/kapa.js docs/js/rating.js docs/js/consent.js docs/js/version-select.js \
          docs/css/kapa.css docs/css/rating.css docs/css/osano.css

RUN mkdocs build -f mkdocs-base.yml -d /out && test -f /out/index.html || (echo "FATAL: no content" && exit 1)

# --- OFFLINE PATCHING (post-build safety net) ---
# Belt-and-braces sweep in case any page slipped in a Google Fonts link or
# analytics snippet the pre-build patch didn't anticipate.
RUN find /out -type f -name "*.html" -exec sed -i -E 's#<link[^>]*fonts\.(googleapis|gstatic)\.com[^>]*>##g' {} +
RUN find /out -type f -name "*.html" -exec sed -i -E 's#<script[^>]*google-analytics[^>]*>.*</script>##g' {} +
RUN chmod -R 755 /out

# --- Stage 2: Serve the Offline Site ---
FROM nginx:alpine

COPY --from=builder /out /usr/share/nginx/html
COPY default.conf /etc/nginx/conf.d/default.conf

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
