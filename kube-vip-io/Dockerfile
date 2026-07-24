# --- Stage 1: Build the Static Site ---
# The kube-vip.io site (kube-vip/website) is a Hugo site using the "compose"
# theme, loaded as a Hugo Module (see go.mod / config.toml). netlify.toml pins
# HUGO_VERSION = 0.147.2, so we match that exactly for a reproducible build.
FROM golang:1.24-bookworm AS builder

WORKDIR /app

# Prerequisites: git (Hugo Modules are fetched via `go`/git under the hood),
# ca-certificates + curl (module + Hugo binary download).
RUN apt-get update && apt-get install -y \
    git \
    ca-certificates \
    curl \
    && rm -rf /var/lib/apt/lists/*

RUN git config --global --add safe.directory '*'

# Install Hugo Extended, pinned to the version used by the upstream
# netlify.toml build (HUGO_VERSION = "0.147.2").
ENV HUGO_VERSION=0.147.2
RUN curl -L "https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-amd64.tar.gz" \
    | tar -xz -C /usr/local/bin hugo

# Copy the kube-vip website source
COPY kube-vip-src/ .

# Vendor the Hugo Module theme ("compose") so the actual `hugo build` step
# below does not need network access.
RUN hugo mod vendor || true

# --- OFFLINE PATCHING ---
# 1. content/manifest/index.html is a raw static tool page (manifest
#    generator) that pulls a Google Font and two cdnjs-hosted scripts
#    (babel-polyfill + js-yaml, the latter is required at runtime by the page
#    to parse/emit YAML). Vendor those two scripts locally and drop the
#    Google Fonts stylesheet.
#
#    It also has two more offline defects, both fixed below:
#      a. Upstream ships this page's stylesheet as content/manifest/style.cc
#         (a typo/bug in kube-vip/website itself — verified against the live
#         repo), while index.html links to "style.css". Hugo copies page
#         resources verbatim, so the misnamed file never satisfies the link
#         and the page 404s on its own CSS. Fix by adding a style.css copy
#         under the name the page actually requests.
#      b. The version-picker <script> does a live
#         fetch('https://api.github.com/repos/kube-vip/kube-vip/releases') to
#         populate the version <select>. That's a hard network dependency
#         with no offline fallback, so neutralize it with an inert Promise —
#         the page still loads and the form still works, it just won't be
#         able to pre-populate the live kube-vip release list (expected
#         offline tradeoff; users can still type a version manually via the
#         rest of the form).
RUN mkdir -p static/vendor && \
    curl -fsSL -o static/vendor/babel-polyfill.min.js \
      https://cdnjs.cloudflare.com/ajax/libs/babel-polyfill/6.13.0/polyfill.min.js && \
    curl -fsSL -o static/vendor/js-yaml.min.js \
      https://cdnjs.cloudflare.com/ajax/libs/js-yaml/4.1.0/js-yaml.min.js && \
    cp content/manifest/style.cc content/manifest/style.css && \
    sed -i -E 's#<link href="https://fonts\.googleapis\.com[^"]*" rel="stylesheet">##g' content/manifest/index.html && \
    sed -i -E "s#https://cdnjs\.cloudflare\.com/ajax/libs/babel-polyfill/6\.13\.0/polyfill\.min\.js#/vendor/babel-polyfill.min.js#g" content/manifest/index.html && \
    sed -i -E "s#https://cdnjs\.cloudflare\.com/ajax/libs/js-yaml/4\.1\.0/js-yaml\.min\.js#/vendor/js-yaml.min.js#g" content/manifest/index.html && \
    sed -i -E "s#fetch\('https://api\.github\.com/repos/kube-vip/kube-vip/releases'\)#Promise.resolve({ json: () => Promise.resolve([]) })#" content/manifest/index.html && \
    test -f content/manifest/style.css && \
    ! grep -q 'api\.github\.com' content/manifest/index.html

# 2. content/_index.md (the homepage) embeds a live GitHub Actions CI-status
#    badge image (github.com/.../badge.svg, linking to github.com/.../actions)
#    — an external runtime image request with no offline equivalent. Drop it.
RUN sed -i -E '/\[!\[Build and publish main image regularly\]\(https:\/\/github\.com\/kube-vip\/kube-vip\/actions/d' content/_index.md && \
    ! grep -q 'github\.com/kube-vip/kube-vip/actions' content/_index.md

# 2b. The "compose" theme also renders a sitewide "view source on GitHub" nav
#     icon (layouts/_partials/nav.html, `{{ with $config.source }}`) driven by
#     config/_default/params.toml's [source] table, pointing at
#     github.com/kube-vip/kube-vip on every single page (home, docs,
#     manifest, ...). It's an inert `<a target="_blank">` link (nothing
#     fetches it on load), but for a fully airgapped mirror we don't want a
#     github.com reference anywhere in the served HTML, so drop the whole
#     [source] table — the theme's `with` guard then omits the nav item
#     entirely rather than rendering it with an empty/dead href.
RUN sed -i -E '/^\[source\]$/,/^$/d' config/_default/params.toml && \
    ! grep -q 'github\.com/kube-vip/kube-vip' config/_default/params.toml

# 3. The "compose" theme's mermaid partial would pull mermaid from jsDelivr,
#    but only if a page's content contains the literal string "mermaid" —
#    verified none of the kube-vip docs pages do, so it never renders. Assert
#    that remains true so a future doc edit can't silently reintroduce it.
RUN ! grep -ril "mermaid" content/ 2>/dev/null | grep -q . || \
      (echo "FATAL: 'mermaid' found in content; jsDelivr mermaid script would render" && exit 1)

# Build the site. baseURL is "/" since it's served from the container root.
RUN hugo --gc --minify --baseURL "/" -d /out && \
    test -f /out/index.html || (echo "FATAL: no content" && exit 1)

# Belt-and-braces: strip any Google Fonts stylesheet links and known
# analytics/CDN script tags that might have slipped through, plus assert the
# manifest page's local style.css made it into the output and that the two
# known external-runtime-request defects (CI badge img on the homepage, live
# releases fetch on the manifest page) didn't slip back in. This deliberately
# does NOT grep the whole /out tree for the bare strings "api.github.com" /
# "github.com" — several docs pages legitimately show them in code examples
# (e.g. `curl https://api.github.com/repos/...`) as illustrative text, not as
# live requests, and that's fine offline.
RUN find /out -type f -name "*.html" -exec sed -i -E 's#<link[^>]*fonts\.(googleapis|gstatic)\.com[^>]*>##g' {} + && \
    find /out -type f -name "*.html" -exec sed -i -E '/googletagmanager\.com|google-analytics\.com/d' {} + && \
    test -f /out/manifest/style.css && \
    ! grep -q 'fetch(.*api\.github\.com' /out/manifest/index.html && \
    ! grep -q 'badge\.svg' /out/index.html

RUN chmod -R 755 /out

# --- Stage 2: Serve the Offline Site ---
FROM nginx:alpine

COPY --from=builder /out /usr/share/nginx/html
COPY default.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
