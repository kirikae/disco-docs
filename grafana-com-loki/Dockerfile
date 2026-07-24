# --- Stage 1: Build the Static Site ---
#
# Grafana Loki's docs (docs/sources in grafana/loki) are written for grafana.com's
# proprietary Hugo theme. Grafana ships the theme + shortcodes as a public Docker
# image, `grafana/docs-base` (see grafana/loki/docs/docs.mk and `make-docs`, which
# normally use this image to run `hugo server` for local live-preview only).
#
# That image *also* contains the real `grafana/website` Makefile, which has a
# `hugo` target that runs a genuine static build:
#   hugo --environment=docs --destination=dist/ ...
# So instead of the live-preview server, we mount Loki's docs/sources into the
# image's content tree and run that `hugo` target directly for a real static
# site.
#
# Scope/version: mounted at content/docs/loki/latest (a real "latest" version,
# not make-docs' own "UNVERSIONED" local-preview convention). This matters:
# with no version segment, the theme's cross-section link/relref rendering
# (partials/docs/virtual-latest.html + version-substitution) mis-resolves —
# verified by building both ways and diffing internal links — producing wrong
# nested paths for many cross-references (e.g. a get-started page linking to
# "/docs/loki/get-started/configure/storage/" instead of
# "/docs/loki/configure/storage/"). Building as version "latest" matches how
# grafana.com actually serves Loki (.../docs/loki/latest/) and also matches
# what Loki's own markdown hardcodes in hrefs/hero links, and the theme
# resolves it correctly. This builds the docs/sources tree from the loki
# repo's default branch only; grafana.com's real multi-version output (latest,
# next, per-release) requires checking out several release branches
# side-by-side the way Grafana's internal CI does — out of scope here.
FROM grafana/docs-base:latest AS builder

WORKDIR /hugo

COPY loki-src/docs/sources /hugo/content/docs/loki/latest

# WEBSITE_SKIP_MOUNTS=true is already set as a default env in this image, which
# skips the `mounts` Makefile target (a Node script that wires up multi-repo
# version mounts) — we don't need it for a single unversioned project.
ENV HUGO_ENV=docs

RUN make hugo

# --- OFFLINE PATCHING ---
# The Hugo build itself is fully local/static, but every page's <head> loads the
# actual theme CSS/JS/fonts live from grafana.com (https://grafana.com/web/*),
# because in normal use grafana.com itself serves those shared assets to every
# product's docs. Vendor the ones actually referenced by the built Loki pages
# (fetched from grafana.com at build time here, nothing committed to the repo),
# and remove everything that is not: this pulls in Grafana's marketing/analytics
# bundle (RudderStack, Hotjar, GA/GTM, Algolia, Crazy Egg, LinkedIn/Facebook
# pixels, Qualtrics, ...) which must never run offline (or at all, in this repo).
RUN set -eux; \
    mkdir -p dist/web/fonts/inter dist/web/fonts/poppins \
             dist/web/fonts/font-awesome/css dist/web/fonts/font-awesome/fonts \
             dist/web/icons/fonts dist/media/docs/loki dist/media/docs/alloy \
             dist/media/docs/icons dist/media/docs/learning-journey \
             dist/media/images/icons dist/media/images/svg; \
    base=https://grafana.com; \
    # Theme stylesheets + the one page-conditional JS bundle (mermaid.js is a
    # fully self-contained, non-tracking, MIT-licensed bundle - safe to keep).
    for f in web/fonts.min.css web/shared.min.css web/app.min.css \
             web/marketing-ui.min.css web/fonts/font-awesome/css/font-awesome.min.css \
             web/mermaid.js; do \
      curl -fsSL -o "dist/$f" "$base/$f"; \
    done; \
    # Fonts (Inter, Poppins) referenced by fonts.min.css via root-relative url().
    for f in Inter-BoldItalic.woff2 Inter-Bold.woff2 Inter-Italic.woff2 \
             Inter-MediumItalic.woff2 Inter-Medium.woff2 Inter-Regular.woff2 \
             Inter-SemiBoldItalic.woff2 Inter-SemiBold.woff2; do \
      curl -fsSL -o "dist/web/fonts/inter/$f" "$base/web/fonts/inter/$f"; \
    done; \
    for w in 300 500 600 700 800; do \
      for sfx in "" italic; do \
        n="poppins-v23-latin-$w$sfx"; \
        curl -fsSL -o "dist/web/fonts/poppins/$n.ttf" "$base/web/fonts/poppins/$n.ttf"; \
        curl -fsSL -o "dist/web/fonts/poppins/$n.woff2" "$base/web/fonts/poppins/$n.woff2"; \
      done; \
    done; \
    curl -fsSL -o dist/web/fonts/poppins/poppins-v23-latin-italic.ttf "$base/web/fonts/poppins/poppins-v23-latin-italic.ttf"; \
    curl -fsSL -o dist/web/fonts/poppins/poppins-v23-latin-italic.woff2 "$base/web/fonts/poppins/poppins-v23-latin-italic.woff2"; \
    curl -fsSL -o dist/web/fonts/poppins/poppins-v23-latin-regular.ttf "$base/web/fonts/poppins/poppins-v23-latin-regular.ttf"; \
    curl -fsSL -o dist/web/fonts/poppins/poppins-v23-latin-regular.woff2 "$base/web/fonts/poppins/poppins-v23-latin-regular.woff2"; \
    # Font Awesome 4 webfont files (referenced relatively from font-awesome.min.css).
    for ext in eot svg ttf woff2 woff; do \
      curl -fsSL -o "dist/web/fonts/font-awesome/fonts/fontawesome-webfont.$ext" \
        "$base/web/fonts/font-awesome/fonts/fontawesome-webfont.$ext"; \
    done; \
    # Grafana icon font used by the docs chrome (menu/toc icons).
    for ext in eot svg ttf woff; do \
      curl -fsSL -o "dist/web/icons/fonts/gicons.$ext" "$base/web/icons/fonts/gicons.$ext"; \
    done; \
    # Content images referenced by Loki pages from grafana.com's media host,
    # which is not part of the docs-base Hugo build output.
    for m in docs/loki/get-started-architecture.png \
             docs/loki/get-started-drill-down-container.png \
             docs/loki/get-started-drill-down.png \
             docs/loki/get-started-flog-v3.png \
             docs/loki/get-started-greenhouse.png \
             docs/loki/getting-started-alloy-ui.png \
             docs/loki/getting-started-loki-stack-3.png \
             docs/loki/grafana-query-builder-v2.png \
             docs/loki/k8s-logs-alloy-ui.png \
             docs/loki/k8s-logs-explore-logs.png \
             docs/loki/k8s-logs-tempo.png \
             docs/loki/logo-grafana-loki.png \
             docs/loki/loki-ile.svg \
             docs/loki/loki-k8s-logs.png \
             docs/alloy/flow-diagram-small-alloy.png \
             docs/icons/docs-play.svg \
             docs/learning-journey/map.svg \
             images/icons/icon-copy-small-2.svg \
             images/svg/thumbs-up.svg \
             docs/loki/loki-log-format.png; do \
      curl -fsSL -o "dist/media/$m" "$base/media/$m"; \
    done; \
    # Favicon + chat-icon: shipped by the real website repo's own static/ dir,
    # not produced by a docs-base build that only mounts Loki's content.
    curl -fsSL -o dist/static/assets/img/fav32.png "$base/static/assets/img/fav32.png"; \
    curl -fsSL -o dist/static/img/grot-chat.svg "$base/static/img/grot-chat.svg"; \
    # Neutralize the handful of absolute https:// url() references left in the
    # vendored CSS (grafanacon/marketing background images unrelated to docs
    # pages) so nothing ever points off-host, even for unmatched selectors.
    sed -i -E 's#url\(https://[a-zA-Z0-9./_-]*\)#url(data:image/gif;base64,R0lGODlhAQABAIAAAAAAAP///ywAAAAAAQABAAACAUwAOw==)#g' \
      dist/web/app.min.css dist/web/marketing-ui.min.css

# Rewrite every built Loki page:
#  - drop Grafana's marketing/analytics bundle entirely (app.js, banner.js,
#    marked.js, marketing-ui.js, Crazy Egg);
#  - point the remaining grafana.com/web/* and grafana.com/media/* asset URLs
#    (stylesheets, mermaid.js, the one absolute-URL content image) at the
#    vendored local copies;
#  - promote every `data-src`/lazy-load image to a real `src` — the theme
#    ships all content images with no `src` at all, only a `data-src` that
#    app.js swaps in on scroll; without app.js every image on every page
#    would otherwise render broken.
# Dropping app.js (an Alpine.js app) also removes the JS that expands nested
# left-nav sections (`display:none` until clicked); a small stylesheet
# override below keeps the whole nav tree permanently expanded so every page
# stays reachable without JS.
RUN find dist/docs/loki/latest -type f -name '*.html' -print0 | xargs -0 sed -i \
      -e 's#<script type=text/javascript src=https://grafana\.com/web/app\.js></script>##g' \
      -e 's#<script type=text/javascript src=https://grafana\.com/web/banner\.js></script>##g' \
      -e 's#<script type=text/javascript src=https://grafana\.com/web/marked\.js></script>##g' \
      -e 's#<script type=text/javascript src=https://grafana\.com/web/marketing-ui\.js></script>##g' \
      -e 's#<script[^>]*crazyegg[^>]*></script>##g' \
      -e 's#data-src=#src=#g' \
      -e 's#https://grafana\.com/web/#/web/#g' \
      -e 's#https://grafana\.com/media/#/media/#g' \
      -e 's#</head>#<style>.docs__menu--sub{display:block!important}</style></head>#' && \
    printf '.docs__menu--sub{display:block!important}\n' >> dist/web/app.min.css

# The LogQL "query analyzer" widget (docs/loki/latest/query/analyzer/) is
# Loki's own content, not the grafana.com theme — but its bundled script.js
# POSTs whatever query the user types to a live backend
# (https://logql-analyzer.grafana.net) on every "Analyze" click. There's no
# static equivalent to vendor, so disable the one network call it makes,
# replacing it with a clear in-UI "not available offline" message (still
# rendered through the page's existing error-handling path).
RUN test -f dist/docs/loki/latest/query/analyzer/script.js && \
    sed -i 's#return fetch(host + "/api/logql-analyze", {#return { status: 501, text: async () => "The LogQL analyzer needs a live connection to Grafana Cloud and is not available in this offline documentation copy." };\n  return fetch(host + "/api/logql-analyze", {#' \
      dist/docs/loki/latest/query/analyzer/script.js && \
    grep -q 'not available in this offline' dist/docs/loki/latest/query/analyzer/script.js && \
    node --check dist/docs/loki/latest/query/analyzer/script.js

# Sanity check: fail the build if the static homepage didn't come out.
RUN test -s dist/docs/loki/latest/index.html

# --- Stage 2: Serve the Offline Site ---
FROM nginx:alpine

# Drop nginx's own default landing page so /index.html doesn't serve it instead
# of a 404 (the real site index lives at /docs/loki/latest/, see default.conf).
RUN rm -f /usr/share/nginx/html/index.html

# Served under /docs/loki/latest/ (matching both the path Loki's own content
# hardcodes in hrefs/aliases, and how grafana.com itself serves it) plus the
# vendored /web/ and /media/ asset trees alongside it.
COPY --from=builder /hugo/dist/docs/loki/latest /usr/share/nginx/html/docs/loki/latest
COPY --from=builder /hugo/dist/web /usr/share/nginx/html/web
COPY --from=builder /hugo/dist/media /usr/share/nginx/html/media
COPY --from=builder /hugo/dist/static /usr/share/nginx/html/static
COPY default.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
