# --- Stage 1: Build the Static Site ---
# strimzi.io (strimzi/strimzi.github.io) is a Jekyll (Ruby) site. Unlike most
# upstream docs projects, the per-release AsciiDoc documentation ("books") for
# every Strimzi/Bridge version is already generated and committed to the site
# repo under docs/**, so no separate AsciiDoc/AsciidoctorJ toolchain or checkout
# of strimzi/strimzi is required -- a plain Jekyll build renders the full,
# all-versions documentation set.
FROM ruby:3.2-slim AS builder

RUN apt-get update && apt-get install -y --no-install-recommends \
      build-essential \
      git \
      ca-certificates \
      wget \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /site
COPY strimzi-src/ .

RUN gem install bundler && bundle install

# --- OFFLINE PATCHING (pre-build), part 1: vendor external assets ---
# The site chrome (every page) pulls jQuery, highlight.js, Font Awesome and the
# "Nunito" Google Font from CDNs, plus Google Analytics and the kapa.ai chat
# widget. The pre-generated AsciiDoc "full" single-page docs committed under
# docs/**, contributing/guide/ and glossary/guide/ additionally bake in their
# own highlight.js (several versions across releases) and Google Fonts
# (Open Sans / Noto Serif / Droid Sans Mono) references directly into the
# static HTML. Fetch one pinned copy of each and rewrite every reference
# (regardless of the specific version baked into a given release) to the
# vendored copy so the built site needs no network access to render.
RUN set -eu; \
    mkdir -p assets/vendor/js assets/vendor/highlightjs \
             assets/vendor/webfonts \
             assets/vendor/fonts/site/files \
             assets/vendor/fonts/asciidoc/files; \
    UA="Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36"; \
    # jQuery (site-scripts.html)
    wget -q -O assets/vendor/js/jquery.min.js \
      "https://ajax.googleapis.com/ajax/libs/jquery/2.1.4/jquery.min.js"; \
    # Mermaid (mermaid-scripts.html)
    wget -q -O assets/vendor/js/mermaid.min.js \
      "https://cdn.jsdelivr.net/npm/mermaid@11.5.0/dist/mermaid.min.js"; \
    # highlight.js: one pinned build covers the site chrome (9.11.0) and every
    # per-release AsciiDoc "full" page (9.12.0/9.15.6/9.18.3), which all use the
    # same hljs.initHighlightingOnLoad() API.
    wget -q -O assets/vendor/highlightjs/highlight.min.js \
      "https://cdnjs.cloudflare.com/ajax/libs/highlight.js/9.18.3/highlight.min.js"; \
    wget -q -O assets/vendor/highlightjs/default.min.css \
      "https://cdnjs.cloudflare.com/ajax/libs/highlight.js/9.18.3/styles/default.min.css"; \
    wget -q -O assets/vendor/highlightjs/github.min.css \
      "https://cdnjs.cloudflare.com/ajax/libs/highlight.js/9.18.3/styles/github.min.css"; \
    # Font Awesome (site-header.html). Same pinned version as upstream, so the
    # existing `integrity` SRI hash on the <link> still matches; only the href
    # is repointed. The CSS's @font-face src rules use paths relative to the
    # CSS file itself (../webfonts/*), and the CSS is vendored one level down
    # at assets/vendor/fontawesome/all.min.css, so the webfonts it resolves to
    # must live at assets/vendor/webfonts/ (NOT assets/vendor/fontawesome/webfonts/)
    # -- mirroring that exact relative layout needs no rewriting of the CSS
    # itself. Each family ships both woff2 (primary) and ttf (fallback), both
    # referenced in the CSS's font-face src list.
    mkdir -p assets/vendor/fontawesome; \
    wget -q -O assets/vendor/fontawesome/all.min.css \
      "https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.7.2/css/all.min.css"; \
    for f in fa-brands-400.woff2 fa-brands-400.ttf \
             fa-regular-400.woff2 fa-regular-400.ttf \
             fa-solid-900.woff2 fa-solid-900.ttf \
             fa-v4compatibility.woff2 fa-v4compatibility.ttf; do \
      wget -q -O "assets/vendor/webfonts/$f" \
        "https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.7.2/webfonts/$f"; \
    done; \
    # Nunito (site chrome font: main.scss + site-header.html)
    wget -q -U "$UA" -O /tmp/nunito.css \
      "https://fonts.googleapis.com/css2?family=Nunito:wght@400;700&display=swap"; \
    for url in $(grep -oE 'https://fonts\.gstatic\.com[^)]+\.woff2' /tmp/nunito.css | sort -u); do \
      wget -q -O "assets/vendor/fonts/site/files/$(basename "$url")" "$url"; \
    done; \
    sed -E 's#https://fonts\.gstatic\.com[^)]+/([^/)]+\.woff2)#files/\1#g' /tmp/nunito.css > assets/vendor/fonts/site/nunito.css; \
    # Open Sans + Noto Serif + Droid Sans Mono: the exact family query baked
    # into every pre-generated AsciiDoc "full" page across all releases.
    wget -q -U "$UA" -O /tmp/asciidoc-fonts.css \
      "https://fonts.googleapis.com/css?family=Open+Sans:300,300italic,400,400italic,600,600italic%7CNoto+Serif:400,400italic,700,700italic%7CDroid+Sans+Mono:400,700"; \
    for url in $(grep -oE 'https://fonts\.gstatic\.com[^)]+\.woff2' /tmp/asciidoc-fonts.css | sort -u); do \
      wget -q -O "assets/vendor/fonts/asciidoc/files/$(basename "$url")" "$url"; \
    done; \
    sed -E 's#https://fonts\.gstatic\.com[^)]+/([^/)]+\.woff2)#files/\1#g' /tmp/asciidoc-fonts.css > assets/vendor/fonts/asciidoc/asciidoc-fonts.css

# --- OFFLINE PATCHING (pre-build), part 2: repoint the site chrome templates ---
# at the vendored copies above, and strip Google Analytics + the kapa.ai widget
# outright (no offline equivalent makes sense for either).
RUN set -eu; \
    sed -i -E 's#href="https://cdnjs\.cloudflare\.com/ajax/libs/font-awesome/[^"]*"#href="/assets/vendor/fontawesome/all.min.css"#' _includes/site-header.html; \
    sed -i -E 's#href="https://cdnjs\.cloudflare\.com/ajax/libs/highlight\.js/[0-9.]+/styles/default\.min\.css"#href="/assets/vendor/highlightjs/default.min.css"#' _includes/site-header.html; \
    sed -i -E 's#href="https://fonts\.googleapis\.com/css2\?family=Nunito[^"]*"#href="/assets/vendor/fonts/site/nunito.css"#' _includes/site-header.html; \
    sed -i '/Global site tag (gtag.js)/d' _includes/site-header.html; \
    sed -i '/googletagmanager\.com\/gtag/d' _includes/site-header.html; \
    sed -i '/^  <script>$/,/^  <\/script>$/d' _includes/site-header.html; \
    sed -i '/kapa\.ai AI search widget/,/^  ><\/script>$/d' _includes/site-header.html; \
    sed -i -E 's#https://ajax\.googleapis\.com/ajax/libs/jquery/2\.1\.4/jquery\.min\.js#/assets/vendor/js/jquery.min.js#' _includes/site-scripts.html; \
    sed -i -E 's#//cdnjs\.cloudflare\.com/ajax/libs/highlight\.js/9\.11\.0/highlight\.min\.js#/assets/vendor/highlightjs/highlight.min.js#' _includes/site-scripts.html; \
    sed -i -E 's#https://cdn\.jsdelivr\.net/npm/mermaid@[0-9.]+/dist/mermaid\.min\.js#/assets/vendor/js/mermaid.min.js#' _includes/mermaid-scripts.html; \
    sed -i -E "s#https://fonts\.googleapis\.com/css\?family=Nunito[^']*#/assets/vendor/fonts/site/nunito.css#" assets/css/main.scss; \
    grep -q "googletagmanager\|kapa.ai\|cdnjs.cloudflare" _includes/site-header.html && \
      { echo "FATAL: external references remain in site-header.html"; exit 1; } || true

RUN bundle exec jekyll build -d /out && test -f /out/index.html || (echo "FATAL: no content" && exit 1)

# --- OFFLINE PATCHING (post-build): the pre-generated AsciiDoc "full" pages ---
# (docs/**, contributing/guide/full.html, glossary/guide/full.html) are static
# HTML committed upstream and copied through by Jekyll unmodified; patch the
# built output directly. Each release baked in whatever highlight.js version
# was current for its Asciidoctor gem at the time (9.12.0/9.15.6/9.18.3 seen
# across the corpus) plus the same Open Sans/Noto Serif/Droid Sans Mono Google
# Fonts query, so match on version-agnostic patterns.
RUN find /out -type f -name "*.html" -exec sed -i -E \
      -e 's#https://cdnjs\.cloudflare\.com/ajax/libs/highlight\.js/[0-9.]+/highlight\.min\.js#/assets/vendor/highlightjs/highlight.min.js#g' \
      -e 's#https://cdnjs\.cloudflare\.com/ajax/libs/highlight\.js/[0-9.]+/styles/github\.min\.css#/assets/vendor/highlightjs/github.min.css#g' \
      -e 's#https://fonts\.googleapis\.com/css\?family=[^"]*#/assets/vendor/fonts/asciidoc/asciidoc-fonts.css#g' \
      {} +

# --- OFFLINE PATCHING (post-build): neutralize YouTube embeds ---
# The homepage hero, the (unused-by-default) video-content-band template and
# many blog posts under _posts/ embed a YouTube <iframe>. Loading any of them
# fires a cascade of requests to youtube.com, ytimg.com, ggpht.com,
# doubleclick.net and gstatic.com -- unacceptable in an airgapped image.
# Replace every such iframe, on every rendered page, with a zero-request link
# to watch the video on the real site. All iframe attributes (width/height/
# title/allow/...) are rendered on a single line in every case observed
# across the source, so a line-oriented sed pass is sufficient; the delimiter
# is `|` because the replacement contains a literal `#` (a numeric HTML
# entity).
RUN find /out -type f -name "*.html" -exec sed -i -E \
      's|<iframe[^>]*src="https://www\.youtube\.com/embed/([A-Za-z0-9_-]+)[^"]*"[^>]*>[[:space:]]*</iframe>|<div class="offline-video-placeholder"><a href="https://www.youtube.com/watch?v=\1" target="_blank" rel="noopener noreferrer">Watch on YouTube \&#8599;</a></div>|g' \
      {} +; \
    ! grep -rIl 'youtube\.com/embed\|<iframe' /out --include="*.html" | grep -q . || \
      { echo "FATAL: youtube iframe(s) remain in /out"; grep -rIl 'youtube\.com/embed\|<iframe' /out --include="*.html"; exit 1; }

RUN chmod -R 755 /out

# --- Stage 2: Serve the Offline Site ---
FROM nginx:alpine

COPY --from=builder /out /usr/share/nginx/html
COPY default.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
