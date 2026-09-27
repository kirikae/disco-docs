# --- Stage 1: Build the Static Site ---
# docs.gitea.com migrated from Docusaurus to Astro + Starlight. The source
# (gitea.com/gitea/docs, not GitHub) is now a pnpm workspace: the site package is
# sites/docs, the markdown lives at the repo root (docs/, versioned_docs/,
# runner-docs/), and the build output lands in sites/docs/dist. There is no
# docusaurus.config.js any more, and `docusaurus` is not an installed binary.
FROM docker.io/library/node:24-bookworm-slim AS builder

WORKDIR /app

# make: upstream drives its build through its own Makefile, and using those
# targets keeps this in step with how the site is actually built. git: the
# prepare step clones awesome-gitea. python3/g++: native modules during install.
RUN apt-get update && apt-get install -y \
    git \
    curl \
    ca-certificates \
    make \
    python3 \
    g++ \
    && rm -rf /var/lib/apt/lists/*

RUN git config --global --add safe.directory '*'

# The repo pins pnpm through package.json's "packageManager" field; corepack
# activates that exact version.
RUN corepack enable

# Copy the Gitea docs source (hosted on gitea.com, not GitHub)
COPY gitea-docs/ .

# upstream's own prepare target: pnpm install, plus cloning awesome-gitea and
# copying its README into docs/ and each versioned_docs/version-1.*/ tree.
RUN make prepare-docs && test -f docs/awesome.md

# Two build settings, both needed, and neither optional:
#
#  - The Astro build of the full matrix dies with a segfault (exit 139) part way
#    through generating the several thousand OpenAPI operation pages. Raising
#    V8's old-space is necessary but is *not* sufficient on its own: verified by
#    building with the heap at 8 GiB and every locale enabled, which still
#    segfaulted.
#  - GITEA_DOCS_LOCALES is upstream's own content-loader filter (documented in
#    packages/content-loader/src/segments.ts), so this is a supported knob rather
#    than something invented here. Narrowing to English is what lets the build
#    finish, and it matches what this image produced before the migration, when
#    the Docusaurus build was invoked with `--locale en-us`. Starlight still emits
#    zh-cn/zh-tw route shells; it is the content matrix that is narrowed.
ENV NODE_OPTIONS="--max-old-space-size=8192"
ENV GITEA_DOCS_LOCALES=en-us

RUN make build && \
    test -f sites/docs/dist/index.html || (echo "FATAL: Astro generated no content" && exit 1)

# Search is served by Pagefind, which indexes at build time into dist/pagefind/
# and needs no network. Upstream swaps it for hosted Algolia DocSearch only when
# PUBLIC_DOCSEARCH_APP_ID/API_KEY are set, which they are not here -- assert the
# local index exists so that a future default flip cannot quietly leave the
# search box calling out to algolia.net.
RUN test -s sites/docs/dist/pagefind/pagefind.js && \
    ! grep -rqE 'algolia(net)?\.(com|net)' sites/docs/dist --include='*.html'

# --- OFFLINE PATCHING (post-build) ---
# astro.config.mjs injects three analytics scripts through Starlight's `head`
# option, onto every one of the ~5,700 generated pages: the Google Tag Manager
# loader, the inline gtag config that follows it, and Plausible. None has an
# offline equivalent, and patching the built HTML is more reliable than trying to
# delete object literals out of the middle of a JS config array.
RUN find sites/docs/dist -type f -name '*.html' -exec sed -i -E \
      -e 's#<script[^>]*src="https://www\.googletagmanager\.com/[^"]*"[^>]*></script>##g' \
      -e 's#<script>window\.dataLayer=window\.dataLayer[^<]*</script>##g' \
      -e 's#<script[^>]*src="https://plausible\.io/[^"]*"[^>]*></script>##g' {} + && \
    ! grep -rqE 'googletagmanager|google-analytics|plausible\.io' sites/docs/dist --include='*.html'

# Mirror every remotely-hosted <img>. The awesome-gitea content pulled in by the
# prepare step above carries a few badges (shields.io, awesome.re and a hosting
# provider's deploy button). The extension comes from the response Content-Type,
# not the URL, because badge URLs carry query strings and often no extension, and
# an SVG served as application/octet-stream will not render in an <img>. URLs are
# rewritten longest-first so one that is a prefix of another cannot corrupt it,
# and a URL that cannot be fetched is still rewritten to its local path, so the
# page degrades to a missing image rather than reaching for the network on load.
RUN set -eu; \
    mkdir -p sites/docs/dist/vendor/img; \
    find sites/docs/dist -type f -name '*.html' -print0 \
      | xargs -0 grep -hoE '<img[^>]*src="?https?://[^" >]+' \
      | grep -oE 'https?://[^" >]+' | sort -u \
      | awk '{ print length, $0 }' | sort -rn | cut -d' ' -f2- > /tmp/ext-img-urls.txt || true; \
    n=0; \
    while read -r url; do \
      [ -n "$url" ] || continue; \
      n=$((n + 1)); \
      ct=$(curl -fsSL -o /tmp/mirror.bin -w '%{content_type}' "$url" || echo unavailable); \
      case "$ct" in \
        image/svg*)  ext=svg ;; \
        image/png*)  ext=png ;; \
        image/jpeg*) ext=jpg ;; \
        image/gif*)  ext=gif ;; \
        image/webp*) ext=webp ;; \
        image/avif*) ext=avif ;; \
        *) ext=img; echo "WARNING: could not mirror ${url} (content-type: ${ct})" ;; \
      esac; \
      name="${n}.${ext}"; \
      cp /tmp/mirror.bin "sites/docs/dist/vendor/img/${name}" 2>/dev/null || true; \
      find sites/docs/dist -type f -name '*.html' -exec sed -i "s#${url}#/vendor/img/${name}#g" {} +; \
    done < /tmp/ext-img-urls.txt; \
    echo "Mirrored ${n} remote image(s)"

# Belt-and-braces sweep for font markup. Starlight ships a system font stack and
# this site adds no webfont, but a future upstream change could.
RUN find sites/docs/dist -type f -name "*.html" -exec sed -i -E 's#<link[^>]*fonts\.(googleapis|gstatic)\.com[^>]*>##g' {} +

# Assert the built site loads nothing over the network: no <script>/<img>/<link>
# pointing at an absolute or protocol-relative URL, and no known asset CDN or
# analytics host anywhere in the served HTML/CSS/JS -- the second is the only way
# to see a URL inside an inline script.
#
# docs.gitea.com itself is excluded from the first check: Starlight emits
# <link rel="canonical">, <link rel="alternate" hreflang=...> and og:url from the
# configured site URL. Those are metadata, not load-time fetches, and pointing
# them at the authoritative site is correct for a mirror.
RUN set -eu; \
    ! find sites/docs/dist -type f \( -name '*.html' -o -name '*.css' \) -print0 \
      | xargs -0 grep -hoE '<(script|img|link|iframe)[^>]*(src|href)=["'"'"']?(https?:)?//[^"'"'"' >]+' \
      | grep -qv 'docs\.gitea\.com'; \
    ! find sites/docs/dist -type f \( -name '*.html' -o -name '*.css' -o -name '*.js' -o -name '*.mjs' \) \
        ! -name '*.map' -print0 \
      | xargs -0 grep -qoE '(\\?/\\?/)(cdn\.jsdelivr\.net|unpkg\.com|cdnjs\.cloudflare\.com|code\.jquery\.com|fonts\.googleapis\.com|fonts\.gstatic\.com|www\.googletagmanager\.com|www\.google-analytics\.com|plausible\.io|img\.shields\.io|[a-z0-9-]*\.algolia(net)?\.(com|net))'

# The hardened nginx image serves as UID 65532, which can only read
# world-readable content. Nothing above guarantees that, so sweep the
# output: a+rX adds read for everyone and traverse on directories only,
# without marking regular files executable.
RUN chmod -R a+rX sites/docs/dist

# --- Stage 2: Serve the Offline Site ---
# Red Hat Hardened Images nginx: distroless (no shell, no package manager) and
# runs as UID 65532, so this stage listens on 8080 (see default.conf), can RUN
# nothing, and must not set a CMD -- the image's own CMD carries the flags nginx
# needs after its /usr/sbin/nginx entrypoint. See README.md for the details and
# for how to verify the signatures on the published images.
FROM registry.access.redhat.com/hi/nginx:latest

COPY --from=builder /app/sites/docs/dist /usr/share/nginx/html
COPY default.conf /etc/nginx/conf.d/default.conf

EXPOSE 8080
