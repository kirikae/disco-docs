# --- Stage 1: Build the Static Site ---
# goharbor/website is a Hugo site (Bulma/Sass + a small Node/PostCSS pipeline).
# Netlify (netlify.toml) pins HUGO_VERSION=0.74.0, so we install that exact
# extended (libsass-enabled) release instead of a generic/newer Hugo image to
# match upstream's build behaviour precisely.
FROM node:20-bookworm-slim AS builder

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends \
    git \
    ca-certificates \
    curl \
    libstdc++6 \
    && rm -rf /var/lib/apt/lists/*

RUN git config --global --add safe.directory '*'

RUN curl -fsSL https://github.com/gohugoio/hugo/releases/download/v0.74.0/hugo_extended_0.74.0_Linux-64bit.tar.gz \
    | tar -xz -C /usr/local/bin hugo

# Copy the Harbor website source (checked out by CI). The working tree's .git
# directory is required: upstream's own `make prepare` step (load-docs.sh)
# does its own git fetch/checkout/stash gymnastics to pull versioned docs
# content, so a plain file copy without history would break it.
COPY harbor-website/ .

# --- OFFLINE PATCHING (source-level, applied before the build) ---
# 1. Drop Google Tag Manager analytics (script + noscript beacon).
RUN sed -i '/{{\/\* Analytics \*\/}}/,/<!-- End Analytics -->/d' layouts/_default/baseof.html && \
    sed -i '/Analytics if (noscript)/,/Analytics if  (noscript)/d' layouts/_default/baseof.html && \
    sed -i '/^\s*<!-- Analytics -->\s*$/d' layouts/_default/baseof.html

# 2. Google Fonts (PT Sans) and Font Awesome are not loaded via <link> tags but
#    baked into the compiled/minified stylesheet as literal CSS `@import
#    url(...)` statements (see assets/sass/style.sass), so they must be
#    repointed at the local copies vendored below rather than stripped from
#    HTML after the fact.
RUN sed -i 's#printf "https://fonts.googleapis.com/css?family=%s" (delimit $fontSlice "|")#"/vendor/fonts/pt-sans.css"#' assets/sass/style.sass && \
    sed -i 's#printf "https://use.fontawesome.com/releases/v%s/css/all.css" .#"/vendor/fontawesome/css/all.min.css"#' assets/sass/style.sass

# 3. Vendor Alpine.js (drives the nav/theme toggles) instead of loading it
#    from jsDelivr.
RUN sed -i -E 's#<script src="https://cdn\.jsdelivr\.net/gh/alpinejs/alpine@v\{\{ \$alpineJsVersion \}\}/dist/alpine-ie11\.js" defer></script>#<script src="/vendor/alpine/alpine-ie11.js" defer></script>#' layouts/partials/javascript.html

# 4. Remove integrations that cannot function offline at all rather than
#    leave dead network calls in the page: Algolia DocSearch is a hosted
#    SaaS search index (no self-hostable equivalent bundled here -- the
#    search box remains in the UI but is inert), and the buttons.github.io
#    widget renders live GitHub star/watch/fork counts. The plain
#    `<a class="github-button">` links it would have enhanced are left in
#    place as ordinary links.
RUN sed -i '/Algolia DocSearch JS/d; /docsearch\.min\.js/d' layouts/partials/javascript.html && \
    sed -i '/^<script type="text\/javascript">$/,/^<\/script>$/d' layouts/partials/javascript.html && \
    sed -i '/docsearch/d' layouts/partials/css.html && \
    sed -i '/buttons\.github\.io/d' layouts/_default/baseof.html && \
    sed -i '/coreinfrastructure\.org/d; /img\.shields\.io/d' layouts/partials/github-info.html

# --- BUILD ---
# `load-docs.sh` (upstream's own docs-loading step, run via `make prepare`)
# fetches every release-X branch plus the harbor-cli repo directly from
# GitHub, so this stage needs network access. The resulting served site has
# zero runtime external requests.
RUN chmod +x load-docs.sh cli-docs.sh && ./load-docs.sh

RUN npm install

RUN hugo --minify --baseURL "/" -d /app/public

# --- VENDOR EXTERNAL ASSETS FETCHED AT BUILD TIME ---
RUN set -eu; \
    npm install --no-save --prefix /opt/fontawesome @fortawesome/fontawesome-free@6.6.0; \
    mkdir -p /app/public/vendor/fontawesome; \
    cp -a /opt/fontawesome/node_modules/@fortawesome/fontawesome-free/css /app/public/vendor/fontawesome/; \
    cp -a /opt/fontawesome/node_modules/@fortawesome/fontawesome-free/webfonts /app/public/vendor/fontawesome/; \
    mkdir -p /app/public/vendor/alpine; \
    curl -fsSL -o /app/public/vendor/alpine/alpine-ie11.js "https://cdn.jsdelivr.net/gh/alpinejs/alpine@v2.1.2/dist/alpine-ie11.js"; \
    UA="Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36"; \
    mkdir -p /app/public/vendor/fonts/files; \
    curl -fsSL -A "$UA" -o /tmp/pt-sans.css "https://fonts.googleapis.com/css?family=PT+Sans:300,400,600,700"; \
    for url in $(grep -oE 'https://fonts\.gstatic\.com[^)]+\.woff2' /tmp/pt-sans.css | sort -u); do \
      curl -fsSL -o "/app/public/vendor/fonts/files/$(basename "$url")" "$url"; \
    done; \
    sed -E 's#https://fonts\.gstatic\.com[^)]+/([^/)]+\.woff2)#files/\1#g' /tmp/pt-sans.css > /app/public/vendor/fonts/pt-sans.css

# Belt-and-braces: strip any stray SRI attributes (in case a resource's
# content changed after it was fingerprinted) and sweep the built HTML for
# any remaining Google/analytics references.
RUN find /app/public -type f -name "*.html" -exec sed -i -E 's/ integrity="sha[0-9]*-[^"]*"//g; s/ crossorigin="anonymous"//g' {} + || true && \
    find /app/public -type f -name "*.html" -exec sed -i -E '/(google-analytics|googletagmanager)/d' {} + || true

# --- Stage 2: Serve the Offline Site ---
FROM nginx:alpine

COPY --from=builder /app/public /usr/share/nginx/html
COPY default.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
