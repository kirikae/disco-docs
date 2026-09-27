# --- Stage 1: Build the Static Site ---
# gateway-api.sigs.k8s.io lives in kubernetes-sigs/gateway-api under site/ and is
# a Hugo site using the Docsy theme, pulled in as a Hugo Module. Reproducing
# `make build-docs-netlify` needs four toolchains, all of which the upstream
# netlify.toml pins: Go (the generators, plus a WASM build), Node (Docsy's
# PostCSS pipeline), Python (the conformance and wizard data generators) and
# Hugo Extended itself. Go leads because it is the only one the build compiles
# with; the other three are installed at exactly the pinned versions.
FROM docker.io/library/golang:1.26-bookworm AS builder

WORKDIR /app

# Versions pinned to site-src's netlify.toml [build.environment] / Makefile.
ARG HUGO_VERSION=0.160.1
ARG NODE_VERSION=20.20.0

# xz-utils: the Node tarball is .tar.xz. git: the API reference generator reads
# the release branches (see below). curl: fetching the toolchains and, later,
# mirroring the conformance badges.
RUN apt-get update && apt-get install -y \
    git \
    curl \
    ca-certificates \
    python3 \
    python3-venv \
    xz-utils \
    && rm -rf /var/lib/apt/lists/*

RUN curl -fsSL "https://nodejs.org/dist/v${NODE_VERSION}/node-v${NODE_VERSION}-linux-x64.tar.xz" \
    | tar -xJ -C /usr/local --strip-components=1

RUN curl -fsSL "https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-amd64.tar.gz" \
    | tar -xz -C /usr/local/bin hugo

RUN git config --global --add safe.directory '*'

COPY gateway-api-src/ .

# `make install-deps`, split in two so the layers cache independently. Debian's
# python3 is 3.11 rather than netlify's 3.12; the generator scripts use nothing
# newer, and a venv keeps pip off the system interpreter (PEP 668).
RUN python3 -m venv /venv && \
    /venv/bin/pip install --no-cache-dir pandas PyYAML semver python-frontmatter tabulate
RUN cd site && npm install

# --- CONTENT GENERATION (upstream's build-docs-netlify prerequisites) ---
# A large share of the site is generated rather than committed, so each of these
# is required for the docs to be complete, not optional polish.

# The enhancement proposals (GEPs) are written under geps/ at the repo root and
# turned into site content, and the implementations list is assembled from
# conformance/list/ plus the submitted conformance reports.
RUN hack/update-geps.sh && test -d site/content/en/geps
RUN hack/update-implist.sh && test -s site/content/en/docs/implementations/list.md

# The API spec reference is generated per supported release: hack/docsy/generate.sh
# reads apis/ and apisx/ out of each of the release-* branches plus main, so the
# site can document v1.4/v1.5/v1.6 and Development side by side.
#
# It reaches those branches with `git fetch "${REMOTE}" <branch>`, defaulting to
# the `origin` remote. Passing the upstream URL instead of relying on origin
# matters for two reasons: the CI checkout is depth-1 with no release branches,
# and actions/checkout leaves a credential header in .git/config -- which, if the
# fetch depended on it, would bake a GitHub token into the registry-backed
# BuildKit cache. The workflow also checks out with persist-credentials: false,
# so the fetch here is plainly anonymous.
ARG UPSTREAM_REMOTE=https://github.com/kubernetes-sigs/gateway-api.git
RUN REMOTE="${UPSTREAM_REMOTE}" hack/docsy/generate.sh && \
    test -s site/content/en/reference/api-spec/main/spec.md

# The controller wizard is a Go program compiled to WebAssembly and driven by a
# generated data file; without both, /wizard/ loads an empty page.
RUN mkdir -p site/static/wizard && \
    GOROOT=$(go env GOROOT); \
    if [ -f "$GOROOT/misc/wasm/wasm_exec.js" ]; then cp -f "$GOROOT/misc/wasm/wasm_exec.js" site/static/wizard/; \
    elif [ -f "$GOROOT/lib/wasm/wasm_exec.js" ]; then cp -f "$GOROOT/lib/wasm/wasm_exec.js" site/static/wizard/; \
    else echo "FATAL: wasm_exec.js not found in GOROOT" && exit 1; fi && \
    GOOS=js GOARCH=wasm go build -o site/static/wizard/main.wasm ./wasm/ && \
    test -s site/static/wizard/main.wasm
RUN mkdir -p site/static/wizard/data && \
    /venv/bin/python3 hack/generate-controller-wizard-data.py --all \
      -o site/static/wizard/data/controller-wizard-data.json && \
    test -s site/static/wizard/data/controller-wizard-data.json

# Per-version conformance tables under docs/implementations/versions/.
RUN /venv/bin/python3 hack/docsy-generate-conformance.py && \
    test -d site/content/en/docs/implementations/versions

# --- OFFLINE PATCHING (pre-build) ---
# Vendor the Docsy theme into site/_vendor so the three CDN URLs hardcoded in
# its templates can be patched. None of them is configurable, so there is no
# config-only way to make a Docsy site self-contained.
RUN cd site && hugo mod vendor && \
    test -d _vendor/github.com/google/docsy/layouts

# Docsy's head.html loads jQuery from code.jquery.com and (with offlineSearch
# enabled, as this site does) lunr from unpkg.com, on every page. Mirror both at
# whatever version the vendored template asks for -- read the URLs out of the
# template rather than hardcoding them, so a Docsy bump cannot leave us serving
# a different version than the site expects -- then repoint the template and
# drop the now-meaningless Subresource Integrity attributes.
RUN set -eu; \
    cd site; \
    head=_vendor/github.com/google/docsy/layouts/_partials/head.html; \
    mkdir -p static/vendor; \
    jq_url=$(grep -oE 'https://code\.jquery\.com/jquery-[0-9.]+\.min\.js' "$head" | head -1); \
    lunr_url=$(grep -oE 'https://unpkg\.com/lunr@[0-9.]+/lunr\.min\.js' "$head" | head -1); \
    test -n "$jq_url" && test -n "$lunr_url"; \
    curl -fsSL -o "static/vendor/$(basename "$jq_url")" "$jq_url"; \
    curl -fsSL -o static/vendor/lunr.min.js "$lunr_url"; \
    sed -i -E \
      -e 's#src="https://code\.jquery\.com/(jquery-[0-9.]+\.min\.js)"#src="/vendor/\1"#' \
      -e 's#src="https://unpkg\.com/lunr@[0-9.]+/lunr\.min\.js"#src="/vendor/lunr.min.js"#' \
      -e '/^[[:space:]]*integrity="sha(256|384|512)-[^"]*"$/d' \
      -e 's#^([[:space:]]*)crossorigin="anonymous"></script>#\1></script>#' \
      "$head"; \
    ! grep -qE 'code\.jquery\.com|unpkg\.com' "$head"; \
    grep -q 'src="/vendor/lunr.min.js"' "$head"

# Docsy renders Mermaid diagrams (used on ~10 pages here, mostly GEPs) with an
# inline <script type="module"> that does `import mermaid from "https://
# cdn.jsdelivr.net/npm/mermaid@latest/..."`. Being an ESM import rather than a
# src attribute, it is easy to miss; being pinned to @latest, it is not even
# reproducible online. The ESM bundle is code-split across a chunks/ directory,
# so instead of mirroring that tree we load Mermaid's self-contained UMD build
# from a plain <script> and delete the import -- `mermaid` then resolves as a
# global, which the rest of upstream's initialization code uses unchanged. The
# classic script is not deferred while module scripts always are, so Mermaid is
# loaded by the time the module body runs.
#
# The partial also validates the CDN URL with resources.GetRemote at build time
# and aborts the build if it 404s; that check goes too, since the URL it probes
# is no longer the one the page uses.
ARG MERMAID_VERSION=11
RUN set -eu; \
    cd site; \
    part=_vendor/github.com/google/docsy/layouts/_partials/scripts/mermaid.html; \
    mkdir -p static/vendor/mermaid; \
    curl -fsSL -o static/vendor/mermaid/mermaid.min.js \
      "https://unpkg.com/mermaid@${MERMAID_VERSION}/dist/mermaid.min.js"; \
    sed -i \
      -e '/resources\.GetRemote \$cdnurl/,/^{{ end -}}$/d' \
      -e 's#^<script type="module" async>#<script src="/vendor/mermaid/mermaid.min.js"></script>\n<script type="module" async>#' \
      -e '/import mermaid from "{{ \$cdnurl }}";/d' \
      "$part"; \
    ! grep -q 'cdn\.jsdelivr\.net' "$part"; \
    grep -q '/vendor/mermaid/mermaid.min.js' "$part"; \
    grep -q 'mermaid.initialize(settings)' "$part"

# Strip the Google Analytics measurement ID from [services.googleAnalytics].
# Hugo only emits the googletagmanager.com gtag.js loader when the ID is set, so
# removing the line removes the script from all ~200 pages. Upstream's own
# comment notes this is the supported way to turn GA off, and that it also
# disables the GA-backed [params.ui.feedback] "was this page helpful" widget.
RUN sed -i -E '/^[[:space:]]*id[[:space:]]*=[[:space:]]*"G-[A-Za-z0-9]+"/d' site/hugo.toml && \
    ! grep -qE '^[[:space:]]*id[[:space:]]*=[[:space:]]*"G-' site/hugo.toml

# Docsy self-hosts its webfonts (Font Awesome ships in the module, and this site
# leaves Docsy's Google Fonts integration off), so there is no font patching to
# do here -- but assert that stays true rather than trusting it, since turning
# $td-enable-google-fonts back on upstream would silently add a
# fonts.googleapis.com stylesheet to every page.
RUN ! grep -rqE 'td-enable-google-fonts[[:space:]]*:[[:space:]]*true' site/assets 2>/dev/null

# baseURL is "/" in hugo.toml since the site is served from the container root.
RUN hugo --source site --minify -d /out --baseURL / && \
    test -f /out/index.html || (echo "FATAL: Hugo generated no content" && exit 1)

# --- OFFLINE PATCHING (post-build) ---
# Mirror every remotely-hosted <img>. The implementations pages render one
# shields.io badge per conformance claim (currently 10 distinct badges), which
# are load-time image requests. The extension is taken from the response
# Content-Type rather than the URL because shields.io URLs carry no extension at
# all, and an SVG served as application/octet-stream will not render in an <img>.
# URLs are rewritten longest-first so one URL that is a prefix of another cannot
# corrupt it, and a URL that cannot be fetched is still rewritten to its local
# path -- the page then shows a missing image instead of reaching for the network.
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

# Assert the built site loads nothing over the network. Two complementary
# checks, because neither alone is sufficient:
#
#   1. No <script>/<img>/<link> points at an absolute or protocol-relative URL.
#      Quotes are optional in the pattern: `hugo --minify` strips them, which is
#      exactly how the jQuery and lunr tags above hid from an earlier version of
#      this check.
#   2. No known asset CDN or analytics host appears anywhere in the served
#      HTML/CSS/JS at all -- including inside inline scripts, where Docsy's
#      Mermaid ESM `import` lived and where check 1 could never have seen it.
#      The alternation also matches a JSON-escaped `https:\/\/`, since minified
#      inline JS spells URLs that way.
#
# Plain <a href> links to github.com, kubernetes.io and the like are left alone
# throughout: they are inert until clicked, and rewriting them would
# misrepresent what the upstream docs say. The single <iframe> embedding the
# community Google Calendar is likewise left in place -- third-party media that
# has no offline equivalent; it renders as an unreachable box.
RUN set -eu; \
    ! find /out -type f \( -name '*.html' -o -name '*.css' \) -print0 \
      | xargs -0 grep -qoE '<(script|img|link)[^>]*(src|href)=["'"'"']?(https?:)?//'; \
    ! find /out -type f \( -name '*.html' -o -name '*.css' -o -name '*.js' -o -name '*.mjs' \) \
        ! -name '*.map' -print0 \
      | xargs -0 grep -qoE '(\\?/\\?/)(cdn\.jsdelivr\.net|unpkg\.com|cdnjs\.cloudflare\.com|code\.jquery\.com|ajax\.googleapis\.com|fonts\.googleapis\.com|fonts\.gstatic\.com|[a-z]+\.bootstrapcdn\.com|use\.fontawesome\.com|kit\.fontawesome\.com|www\.googletagmanager\.com|www\.google-analytics\.com|img\.shields\.io|platform\.twitter\.com|[a-z.]*gravatar\.com)'

RUN chmod -R 755 /out

# --- Stage 2: Serve the Offline Site ---
# Red Hat Hardened Images nginx: distroless (no shell, no package manager) and
# runs as UID 65532, so this stage listens on 8080 (see default.conf), can RUN
# nothing, and must not set a CMD -- the image's own CMD carries the flags nginx
# needs after its /usr/sbin/nginx entrypoint. See README.md for the details and
# for how to verify the signatures on the published images.
FROM registry.access.redhat.com/hi/nginx:latest

COPY --from=builder /out /usr/share/nginx/html
COPY default.conf /etc/nginx/conf.d/default.conf

EXPOSE 8080
