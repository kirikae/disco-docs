# --- Stage 1: Build the Static Site ---
# ovn-kubernetes.io is an MkDocs Material site living at the root of
# ovn-kubernetes/ovn-kubernetes (mkdocs.yml + docs/ + requirements.txt).
FROM python:3.12-slim AS builder

WORKDIR /app

# git: the git-revision-date-localized plugin reads commit dates out of the
# checkout. curl: vendoring the handful of runtime assets Material would
# otherwise pull from a CDN (see the post-build stage below).
RUN apt-get update && apt-get install -y git curl ca-certificates && rm -rf /var/lib/apt/lists/*
RUN git config --global --add safe.directory '*'

COPY ovn-kubernetes-src/ .

RUN pip install --no-cache-dir -r requirements.txt

# --- OFFLINE PATCHING (pre-build) ---
# 1. Disable Material's Google Fonts auto-loading (falls back to system fonts,
#    otherwise the theme injects <link> tags pointing at fonts.googleapis.com).
RUN sed -i '/^theme:/a\  font: false' mkdocs.yml && grep -q 'font: false' mkdocs.yml

# 2. Strip site_url so every generated asset/nav link is relative to the page
#    (the site is served from the container root, not from
#    https://www.ovn.org/ovn-kubernetes/), and strip repo_url/repo_name/edit_uri
#    so mkdocs-material never renders the header "source facts" widget
#    (star/fork/release counts), whose JS makes runtime fetch() calls to
#    api.github.com.
RUN sed -i -E '/^(site_url|repo_url|repo_name|edit_uri):/d' mkdocs.yml && \
    ! grep -qE '^(site_url|repo_url|repo_name|edit_uri):' mkdocs.yml

# 3. Drop the whole `extra:` block, which upstream uses only to declare the
#    mike version provider. Material renders its version selector whenever
#    config.extra.version is truthy, and the selector's JS fetches
#    ../versions.json -- a file only `mike deploy` produces. We build a single
#    version (the tip of the default branch), so there is nothing to select.
#    The sed deletes `extra:` plus every following indented/blank line, leaving
#    the next top-level key (extra_css:) intact.
RUN sed -i '/^extra:$/,/^[^[:space:]]/{/^extra:$/d; /^[^[:space:]]/!d}' mkdocs.yml && \
    ! grep -q 'provider: mike' mkdocs.yml && \
    grep -q '^extra_css:' mkdocs.yml

# 4. Drop the mike and mermaid2 plugins.
#    - mike: single-version build, see above.
#    - mermaid2: its only job here is injecting a <script type="module">
#      importing mermaid from unpkg.com. The diagram markup it emits via
#      superfences (<pre class="mermaid"><code>) is byte-identical to what
#      Material's own native Mermaid support consumes, so dropping the plugin
#      leaves the diagrams rendering through Material's loader instead -- which
#      we repoint at a vendored copy below. The markdown_extensions entry
#      (`!!python/name:mermaid2.fence_mermaid_custom`) keeps working: it only
#      needs the mermaid2 package importable, not the plugin enabled.
RUN sed -i -E '/^  - (mike|mermaid2)$/d' mkdocs.yml && \
    ! grep -qE '^  - (mike|mermaid2)$' mkdocs.yml

RUN mkdocs build -d /out && \
    test -f /out/index.html || (echo "FATAL: MkDocs generated no content" && exit 1)

# --- OFFLINE PATCHING (post-build) ---
# 5. Vendor the two runtime assets Material's JS bundle lazy-loads from
#    unpkg.com and repoint the bundle at the local copies:
#      - Mermaid, injected as a <script> the first time a page containing a
#        `pre.mermaid` block is shown (8 pages do, across okeps/ and features/).
#      - resize-observer-polyfill, injected only when the browser has no
#        ResizeObserver. Vendored too so even an old browser stays offline.
#    Both are fetched with the exact specifier the bundle asks for, so the
#    vendored copy is the same build the online site would have loaded.
RUN set -eu; \
    mkdir -p /out/vendor/mermaid /out/vendor/resize-observer-polyfill; \
    curl -fsSL -o /out/vendor/mermaid/mermaid.min.js \
      https://unpkg.com/mermaid@11/dist/mermaid.min.js; \
    curl -fsSL -o /out/vendor/resize-observer-polyfill/ResizeObserver.js \
      https://unpkg.com/resize-observer-polyfill; \
    find /out/assets/javascripts -name 'bundle.*.min.js' -exec sed -i \
      -e 's#https://unpkg\.com/mermaid@11/dist/mermaid\.min\.js#/vendor/mermaid/mermaid.min.js#g' \
      -e 's#https://unpkg\.com/resize-observer-polyfill#/vendor/resize-observer-polyfill/ResizeObserver.js#g' {} +; \
    ! grep -rq 'unpkg\.com' /out/assets/javascripts/bundle.*.min.js

# 6. Mirror the Twemoji SVGs. pymdownx.emoji's `to_svg` generator renders every
#    emoji shortcode in the docs as <img class="twemoji" src="https://cdn.
#    jsdelivr.net/gh/jdecked/twemoji@<ver>/assets/svg/<codepoint>.svg"> -- a
#    real load-time image request per emoji. Collect whatever the build actually
#    referenced (currently two: warning sign and hundred points), mirror those
#    files, and rewrite the URLs. Driving this off the built HTML rather than a
#    fixed list means new emoji in the docs get mirrored automatically.
RUN set -eu; \
    mkdir -p /out/vendor/twemoji; \
    grep -rhoE 'https://cdn\.jsdelivr\.net/gh/[A-Za-z0-9_.-]+/twemoji@[0-9.]+/assets/svg/[0-9a-f-]+\.svg' \
      /out --include='*.html' | sort -u > /tmp/twemoji-urls.txt || true; \
    while read -r url; do \
      [ -n "$url" ] || continue; \
      curl -fsSL -o "/out/vendor/twemoji/$(basename "$url")" "$url" \
        || echo "WARNING: could not mirror ${url}"; \
    done < /tmp/twemoji-urls.txt; \
    find /out -type f -name '*.html' -exec sed -i -E \
      's#https://cdn\.jsdelivr\.net/gh/[A-Za-z0-9_.-]+/twemoji@[0-9.]+/assets/svg/([0-9a-f-]+\.svg)#/vendor/twemoji/\1#g' {} +; \
    ! grep -rq 'cdn\.jsdelivr\.net' /out --include='*.html'

# 7. Mirror every remotely-hosted <img>. The blog's docs/blog/.authors.yml
#    gives each author an avatar on github.com / raw.githubusercontent.com, and
#    Material emits those URLs verbatim into the blog index, the archive pages
#    and every post -- load-time image requests on the most-visited pages of the
#    site. Rather than hard-coding today's five authors, mirror whatever image
#    URLs the built HTML actually references, so a new blog author or a remote
#    image dropped into a doc is picked up by the next build.
#
#    The extension comes from the response Content-Type, not the URL: plenty of
#    image URLs carry no extension (a shields.io badge, for one) and an SVG
#    served as application/octet-stream will not render in an <img>. URLs are
#    rewritten longest-first so one URL that is a prefix of another cannot
#    corrupt it, and a URL that cannot be fetched is still rewritten to its local
#    path -- the page then shows a missing image instead of reaching for the
#    network on every load.
RUN set -eu; \
    mkdir -p /out/vendor/img; \
    find /out -type f -name '*.html' -print0 \
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
      name="${n}-$(basename "$url" | tr -cd 'A-Za-z0-9._-' | cut -c1-48).${ext}"; \
      cp /tmp/mirror.bin "/out/vendor/img/${name}" 2>/dev/null || true; \
      find /out -type f -name '*.html' -exec sed -i "s#${url}#/vendor/img/${name}#g" {} +; \
    done < /tmp/ext-img-urls.txt; \
    echo "Mirrored ${n} remote image(s)"

# 8. Belt-and-braces sweep for any font/analytics markup that survived the
#    pre-build patches.
RUN find /out -type f -name "*.html" -exec sed -i -E 's#<link[^>]*fonts\.(googleapis|gstatic)\.com[^>]*>##g' {} + && \
    find /out -type f -name "*.html" -exec sed -i -E '/(googletagmanager|google-analytics)\.com/d' {} +

# 9. Assert the built site loads nothing over the network. Three checks, because
#    no single one of them is sufficient:
#
#      a. No <script>/<img>/<link> points at an absolute or protocol-relative
#         URL. Quotes are optional in the pattern because the minify plugin
#         strips them.
#      b. No known asset CDN or analytics host appears anywhere in the served
#         HTML/CSS/JS -- including inside inline scripts, which check (a) cannot
#         see. This is where a theme's lazily-imported ESM module would hide.
#         The alternation also matches a JSON-escaped `https:\/\/`, the form
#         minified inline JS uses. Material's JS bundle is excluded from this
#         check by name: it still *contains* the api.github.com URLs behind the
#         "source facts" widget, which check (c) proves is never rendered.
#      c. Material's source-facts widget markup is absent, so the api.github.com
#         code path in the bundle stays dead. With repo_url stripped (step 2) it
#         is, and this fails the build if the markup ever comes back.
RUN set -eu; \
    ! find /out -type f \( -name '*.html' -o -name '*.css' \) -print0 \
      | xargs -0 grep -qoE '<(script|img|link)[^>]*(src|href)=["'"'"']?(https?:)?//'; \
    ! find /out -type f \( -name '*.html' -o -name '*.css' -o -name '*.js' -o -name '*.mjs' \) \
        ! -name '*.map' ! -name 'bundle.*.min.js' -print0 \
      | xargs -0 grep -qoE '(\\?/\\?/)(cdn\.jsdelivr\.net|unpkg\.com|cdnjs\.cloudflare\.com|code\.jquery\.com|ajax\.googleapis\.com|fonts\.googleapis\.com|fonts\.gstatic\.com|[a-z]+\.bootstrapcdn\.com|use\.fontawesome\.com|kit\.fontawesome\.com|www\.googletagmanager\.com|www\.google-analytics\.com|img\.shields\.io|platform\.twitter\.com|[a-z.]*gravatar\.com)'; \
    ! find /out -type f -name '*.html' -print0 | xargs -0 grep -q 'md-component=source'

RUN chmod -R 755 /out

# --- Stage 2: Serve the Offline Site ---
FROM nginx:alpine

COPY --from=builder /out /usr/share/nginx/html
COPY default.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
