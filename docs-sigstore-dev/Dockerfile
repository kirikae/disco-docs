# --- Stage 1: Build the Hugo (Doks theme) site ---
FROM node:20-bookworm-slim AS builder

WORKDIR /site

RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates \
    git \
    curl \
    && rm -rf /var/lib/apt/lists/*

# The upstream sigstore/docs repo *is* a fork of the Doks Hugo theme scaffold
# (not a separate theme mount / Hugo Module) -- content and theme layouts live
# in the same tree. Hugo itself is fetched by the `postinstall` npm script
# (hugo-installer), pinned to package.json's `otherDependencies.hugo` version,
# and search/diagram/math assets (flexsearch, katex, mermaid, @hyas/images)
# are wired in purely via local `[[module.mounts]]` pointing at node_modules --
# there is no Hugo Modules/go.mod remote-import step required.
COPY sigstore-src/ .

RUN npm ci

# --- OFFLINE PATCHING ---
# Upstream already self-hosts everything needed for a fully offline build:
#   - fonts (Jost) are vendored under static/fonts/vendor/jost and referenced
#     by relative/absURL paths, not Google Fonts
#   - all JS (bootstrap, highlight.js, flexsearch, mermaid, katex, lazysizes,
#     clipboard, instant.page, darkmode) is bundled locally via Hugo Pipes
#     (`resources.Get` against local files / node_modules mounts), never a CDN
#     <script src> -- confirmed no fonts.googleapis/gstatic, no CDN <script>
#     tags, no analytics (gtag/GA/GTM), no Algolia/DocSearch/kapa.ai, and no
#     api.github.com calls anywhere in layouts/ or assets/
#   - math rendering (kaTex) is disabled site-wide in config/_default/params.toml
#     (`kaTex = false`), so its font/JS bundle is skipped entirely by the theme
#   - no content page uses the `{{< video >}}` shortcode (no YouTube/Vimeo iframes)
# So no post-build sed patching of external hosts is required; we only rebase
# the site to a relative baseURL for local/offline serving.
RUN node_modules/.bin/hugo/hugo --gc --minify --baseURL / --destination /out && \
    test -f /out/index.html || (echo "FATAL: no content built" && exit 1)

# The nav-logo partial emits the logo via absURL, which under `--baseURL /`
# renders as the protocol-relative `//sigstore-logo.png` -- browsers read that
# as the host `sigstore-logo.png` (DNS lookup, offline failure). The asset
# actually lives at `/sigstore-logo.png`; rewrite the double slash to a single
# root-relative slash so it loads locally. Guard that no `//sigstore-logo.png`
# survives.
RUN find /out -type f -name "*.html" -exec sed -i 's#//sigstore-logo\.png#/sigstore-logo.png#g' {} + && \
    ! grep -rq '//sigstore-logo\.png' /out || (echo "FATAL: protocol-relative logo ref survived" && exit 1)

RUN chmod -R 755 /out

# --- Stage 2: Serve ---
FROM nginx:alpine

COPY --from=builder /out /usr/share/nginx/html
COPY default.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
