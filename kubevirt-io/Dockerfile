# kubevirt.io is two separate upstream repositories published under one
# hostname, and the site is only coherent with both:
#
#   kubevirt/kubevirt.github.io  ->  /            (Jekyll: home, blogs, labs,
#                                                  videos, gallery, community)
#   kubevirt/user-guide          ->  /user-guide/ (MkDocs Material: the actual
#                                                  KubeVirt documentation)
#
# The Jekyll site has no documentation of its own -- its "Docs" nav entry is a
# hardcoded link to //kubevirt.io/user-guide -- so mirroring only the first
# would leave every documentation link in the image dead. Both are therefore
# built here and assembled into a single image laid out exactly like the live
# site, which also means the cross-links between them keep working untouched.

# --- Stage 1: Build the User Guide (MkDocs Material) ---
FROM python:3.12-slim AS guide-builder

WORKDIR /app

RUN apt-get update && apt-get install -y curl ca-certificates && rm -rf /var/lib/apt/lists/*

COPY kubevirt-user-guide-src/ .

# The user guide has no requirements file; upstream installs this exact set from
# its netlify.toml build command.
RUN pip install --no-cache-dir mkdocs mkdocs-material mkdocs-awesome-nav mkdocs-redirects

# --- OFFLINE PATCHING (pre-build) ---
# 1. Disable Material's Google Fonts auto-loading (falls back to system fonts,
#    otherwise the theme injects <link> tags pointing at fonts.googleapis.com).
RUN sed -i '/^theme:/a\  font: false' mkdocs.yml && grep -q 'font: false' mkdocs.yml

# 2. Strip site_url so every asset and nav link is relative to the page. The
#    guide is served from /user-guide/ here, the same prefix as upstream, but
#    relative links keep it working wherever the image is mounted.
RUN sed -i -E '/^site_url:/d' mkdocs.yml && ! grep -qE '^site_url:' mkdocs.yml

# 3. Strip edit_uri. With it set, Material's content.action.edit/view features
#    render "edit this page" / "view source" links to github.com on every page.
#    They are inert anchors rather than load-time requests, but an airgapped
#    mirror should not offer links it knows cannot resolve; dropping edit_uri
#    makes Material omit the buttons entirely.
RUN sed -i -E '/^edit_uri:/d' mkdocs.yml && ! grep -qE '^edit_uri:' mkdocs.yml

RUN mkdocs build -d /guide && \
    test -f /guide/index.html || (echo "FATAL: MkDocs generated no user guide content" && exit 1)

# --- OFFLINE PATCHING (post-build) ---
# 4. Vendor the two assets Material's JS bundle lazy-loads from unpkg.com.
#    The guide has no Mermaid diagrams today, but the loader ships in the bundle
#    regardless and would fire the moment one is added, so repoint it anyway.
#    resize-observer-polyfill is injected only on browsers without
#    ResizeObserver. Paths are relative to the guide root, which is served at
#    /user-guide/.
RUN set -eu; \
    mkdir -p /guide/vendor/mermaid /guide/vendor/resize-observer-polyfill; \
    curl -fsSL -o /guide/vendor/mermaid/mermaid.min.js \
      https://unpkg.com/mermaid@11/dist/mermaid.min.js; \
    curl -fsSL -o /guide/vendor/resize-observer-polyfill/ResizeObserver.js \
      https://unpkg.com/resize-observer-polyfill; \
    find /guide/assets/javascripts -name 'bundle.*.min.js' -exec sed -i \
      -e 's#https://unpkg\.com/mermaid@11/dist/mermaid\.min\.js#/user-guide/vendor/mermaid/mermaid.min.js#g' \
      -e 's#https://unpkg\.com/resize-observer-polyfill#/user-guide/vendor/resize-observer-polyfill/ResizeObserver.js#g' {} +; \
    ! grep -rq 'unpkg\.com' /guide/assets/javascripts/bundle.*.min.js

# 5. Mirror the Twemoji SVGs. pymdownx.emoji's `to_svg` generator renders every
#    emoji shortcode as <img class="twemoji" src="https://cdn.jsdelivr.net/gh/
#    jdecked/twemoji@<ver>/assets/svg/<codepoint>.svg"> -- a load-time image
#    request per emoji. Driven off the built HTML so new emoji are picked up.
RUN set -eu; \
    mkdir -p /guide/vendor/twemoji; \
    find /guide -type f -name '*.html' -print0 \
      | xargs -0 grep -hoE 'https://cdn\.jsdelivr\.net/gh/[A-Za-z0-9_.-]+/twemoji@[0-9.]+/assets/svg/[0-9a-f-]+\.svg' \
      | sort -u > /tmp/twemoji-urls.txt || true; \
    while read -r url; do \
      [ -n "$url" ] || continue; \
      curl -fsSL -o "/guide/vendor/twemoji/$(basename "$url")" "$url" \
        || echo "WARNING: could not mirror ${url}"; \
    done < /tmp/twemoji-urls.txt; \
    find /guide -type f -name '*.html' -exec sed -i -E \
      's#https://cdn\.jsdelivr\.net/gh/[A-Za-z0-9_.-]+/twemoji@[0-9.]+/assets/svg/([0-9a-f-]+\.svg)#/user-guide/vendor/twemoji/\1#g' {} +

# 6. Mirror every remotely-hosted <img> (today: one asciinema cast thumbnail).
#    The extension comes from the response Content-Type, not the URL, since many
#    image URLs carry no extension and an SVG served as application/octet-stream
#    will not render in an <img>. URLs are rewritten longest-first so one that is
#    a prefix of another cannot corrupt it, and a URL that cannot be fetched is
#    still rewritten to its local path -- the page then degrades to a missing
#    image rather than reaching for the network on load.
RUN set -eu; \
    mkdir -p /guide/vendor/img; \
    find /guide -type f -name '*.html' -print0 \
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
      cp /tmp/mirror.bin "/guide/vendor/img/${name}" 2>/dev/null || true; \
      find /guide -type f -name '*.html' -exec sed -i "s#${url}#/user-guide/vendor/img/${name}#g" {} +; \
    done < /tmp/ext-img-urls.txt; \
    echo "Mirrored ${n} remote image(s) in the user guide"

# 7. Assert the guide loads nothing over the network. Three checks, since none
#    alone is sufficient: no <script>/<img>/<link> points at an absolute or
#    protocol-relative URL (quotes optional -- a minifying theme strips them);
#    no known asset CDN or analytics host appears anywhere in the served
#    HTML/CSS/JS, which is the only way to catch a URL inside an inline script
#    such as a lazily-imported ESM module (Material's JS bundle is excluded by
#    name, as it still contains the api.github.com URLs behind the "source facts"
#    widget); and that widget's markup is absent, so that code path stays dead --
#    it is, since the guide's mkdocs.yml sets no repo_url.
RUN set -eu; \
    ! find /guide -type f \( -name '*.html' -o -name '*.css' \) -print0 \
      | xargs -0 grep -qoE '<(script|img|link)[^>]*(src|href)=["'"'"']?(https?:)?//'; \
    ! find /guide -type f \( -name '*.html' -o -name '*.css' -o -name '*.js' -o -name '*.mjs' \) \
        ! -name '*.map' ! -name 'bundle.*.min.js' -print0 \
      | xargs -0 grep -qoE '(\\?/\\?/)(cdn\.jsdelivr\.net|unpkg\.com|cdnjs\.cloudflare\.com|code\.jquery\.com|ajax\.googleapis\.com|fonts\.googleapis\.com|fonts\.gstatic\.com|[a-z]+\.bootstrapcdn\.com|use\.fontawesome\.com|kit\.fontawesome\.com|www\.googletagmanager\.com|www\.google-analytics\.com|img\.shields\.io|platform\.twitter\.com|[a-z.]*gravatar\.com|www\.redhat\.com/ma)'; \
    ! find /guide -type f -name '*.html' -print0 | xargs -0 grep -q 'md-component=source'

RUN chmod -R 755 /guide

# --- Stage 2: Build the Main Site (Jekyll) ---
# Upstream builds this in a Fedora image with the system Ruby (see
# kubevirt/project-infra images/kubevirt-kubevirt.github.io/Dockerfile). There
# is no committed Gemfile.lock -- the Makefile deletes it -- so the gem set is
# resolved at build time either way; a plain ruby base image with a C toolchain
# for the native extensions is equivalent and much smaller.
FROM docker.io/library/ruby:3.3-slim-bookworm AS site-builder

WORKDIR /app

RUN apt-get update && apt-get install -y \
    build-essential \
    git \
    curl \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

RUN git config --global --add safe.directory '*'

COPY kubevirt-src/ .

RUN bundle install

# --- RELEASE CHANGELOG POSTS ---
# Only five release posts are committed to _posts/releases/; every release from
# v0.8.0 on is generated at build time by scripts/update_changelog.sh, which
# reads the annotated tag messages of kubevirt/kubevirt. Skipping it would leave
# a decade-sized hole in the blog's "releases" category, so it is run here --
# with three edits that keep it cheap and offline-safe:
#
#   1. Clone blobless and without a checkout. The script only ever reads tag
#      messages, so none of the ~1GB of file content is needed; this brings the
#      clone down to ~45MB.
#   2. Drop the `checkout main` / `pull --tags` lines. A --no-checkout clone has
#      no worktree to check out, and the clone already carries every tag.
#   3. `git show` -> `git show --no-patch`. Plain `git show` computes a diff,
#      which in a blobless clone triggers a lazy fetch of the very blobs step 1
#      avoided. --no-patch prints just the tag/commit message -- all the script
#      greps for anyway ("Released on:" date and the "- [PR #...]" change list).
RUN sed -i \
    -e 's#git clone https://github\.com/kubevirt/kubevirt\.git build/kubevirt/#git clone --filter=blob:none --no-checkout https://github.com/kubevirt/kubevirt.git build/kubevirt/#' \
    -e 's#git -C build/kubevirt show \$1#git -C build/kubevirt show --no-patch $1#g' \
    scripts/update_changelog.sh && \
    sed -i -E '/^git -C build\/kubevirt (checkout main|pull --tags)$/d' scripts/update_changelog.sh && \
    grep -q 'filter=blob:none' scripts/update_changelog.sh && \
    grep -q 'show --no-patch' scripts/update_changelog.sh && \
    ! grep -qE '^git -C build/kubevirt (checkout|pull)' scripts/update_changelog.sh

RUN ./scripts/update_changelog.sh && \
    test "$(ls _posts/releases | wc -l)" -gt 5 || \
      (echo "FATAL: changelog generation produced no release posts" && exit 1)

# --- OFFLINE PATCHING (pre-build) ---
# Strip the two third-party scripts that exist only to phone home and have no
# offline equivalent:
#   - www.redhat.com/ma/dpal.js is Adobe Analytics, loaded in <head> on every
#     page.
#   - platform.twitter.com/widgets.js upgrades the <blockquote class=
#     "twitter-tweet"> embeds in a few posts into rendered tweets. Without it
#     the blockquote stays visible as plain text with a link, which is the
#     better offline outcome anyway.
# Also drop the google-site-verification <meta>, which is meaningless off the
# public internet.
RUN sed -i -E \
      -e 's#<script[^>]*//www\.redhat\.com/ma/dpal\.js[^>]*></script>##g' \
      -e '/name="google-site-verification"/d' \
      _includes/head.html && \
    ! grep -q 'dpal\.js' _includes/head.html && \
    ! grep -q 'google-site-verification' _includes/head.html

RUN grep -rlE 'platform\.twitter\.com/widgets\.js' _posts _includes _layouts pages 2>/dev/null \
      | xargs -r sed -i -E 's#<script[^>]*platform\.twitter\.com/widgets\.js[^>]*></script>##g'; \
    ! grep -rq 'platform\.twitter\.com/widgets\.js' _posts _includes _layouts pages

# js/kubevirt-io.js -- which every page loads -- also builds the same widgets.js
# <script> at runtime via createElement/insertBefore, so stripping script tags
# from the templates above does not remove it. Drop the whole `window.twttr`
# bootstrap IIFE; nothing else in the repo references window.twttr. The
# setTimeout block after it, which restyles the Twitter timeline iframe, is left
# alone: with no widget to render it simply finds nothing and does nothing.
RUN sed -i '/^window\.twttr = (function(d, s, id) {$/,/^}(document, "script", "twitter-wjs"));$/d' \
      js/kubevirt-io.js && \
    ! grep -q 'platform\.twitter\.com' js/kubevirt-io.js && \
    ! grep -q 'window\.twttr' js/kubevirt-io.js && \
    grep -q 'collapseNavbar' js/kubevirt-io.js

# Build the site. --future matches upstream's own build (netlify.toml patches it
# in) so posts dated slightly ahead of the build clock are still published.
RUN bundle exec jekyll build --future -d /out && \
    test -f /out/index.html || (echo "FATAL: Jekyll generated no content" && exit 1)

# --- OFFLINE PATCHING (post-build): vendor the CDN assets ---
# The Jekyll templates load their entire front-end from public CDNs. Mirror each
# one at the exact version the templates ask for, preserving the directory shape
# where a stylesheet resolves companions relatively:
#   - Font Awesome 6's all.min.css pulls ../webfonts/fa-*.{woff2,ttf}, so its
#     css/ and webfonts/ directories must stay siblings.
#   - PhotoSwipe's default-skin.min.css pulls default-skin.png, default-skin.svg
#     and preloader.gif from its own directory.
#   - Bootstrap, PhotoSwipe's core CSS, Galleria's "twelve" theme (which inlines
#     its images as data: URIs) and Font Awesome 5's JS kit are self-contained.
RUN set -eu; \
    mkdir -p /out/vendor/js /out/vendor/css \
             /out/vendor/fontawesome/css /out/vendor/fontawesome/webfonts \
             /out/vendor/photoswipe-default-skin; \
    dl() { curl -fsSL -o "$1" "$2"; }; \
    dl /out/vendor/js/jquery.min.js            https://code.jquery.com/jquery.min.js; \
    dl /out/vendor/js/jquery-3.3.1.slim.min.js https://code.jquery.com/jquery-3.3.1.slim.min.js; \
    dl /out/vendor/js/popper.min.js            https://cdnjs.cloudflare.com/ajax/libs/popper.js/1.14.0/umd/popper.min.js; \
    dl /out/vendor/js/bootstrap.min.js         https://stackpath.bootstrapcdn.com/bootstrap/4.1.0/js/bootstrap.min.js; \
    dl /out/vendor/js/fontawesome-all.js       https://use.fontawesome.com/releases/v5.1.0/js/all.js; \
    dl /out/vendor/js/galleria.min.js          https://cdn.jsdelivr.net/npm/galleria@1.6.1/dist/galleria.min.js; \
    dl /out/vendor/js/galleria.twelve.min.js   https://cdn.jsdelivr.net/npm/galleria@1.6.1/dist/themes/twelve/galleria.twelve.min.js; \
    dl /out/vendor/js/photoswipe.min.js        https://cdnjs.cloudflare.com/ajax/libs/photoswipe/4.1.3/photoswipe.min.js; \
    dl /out/vendor/js/photoswipe-ui-default.min.js https://cdnjs.cloudflare.com/ajax/libs/photoswipe/4.1.3/photoswipe-ui-default.min.js; \
    dl /out/vendor/css/bootstrap.min.css       https://stackpath.bootstrapcdn.com/bootstrap/4.1.0/css/bootstrap.min.css; \
    dl /out/vendor/css/photoswipe.min.css      https://cdnjs.cloudflare.com/ajax/libs/photoswipe/4.1.3/photoswipe.min.css; \
    dl /out/vendor/css/galleria.twelve.min.css https://cdn.jsdelivr.net/npm/galleria@1.6.1/dist/themes/twelve/galleria.twelve.min.css; \
    dl /out/vendor/fontawesome/css/all.min.css https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.5.2/css/all.min.css; \
    for f in fa-brands-400 fa-regular-400 fa-solid-900 fa-v4compatibility; do \
      for ext in woff2 ttf; do \
        dl "/out/vendor/fontawesome/webfonts/${f}.${ext}" \
           "https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.5.2/webfonts/${f}.${ext}"; \
      done; \
    done; \
    dl /out/vendor/photoswipe-default-skin/default-skin.min.css https://cdnjs.cloudflare.com/ajax/libs/photoswipe/4.1.3/default-skin/default-skin.min.css; \
    for f in default-skin.png default-skin.svg preloader.gif; do \
      dl "/out/vendor/photoswipe-default-skin/${f}" \
         "https://cdnjs.cloudflare.com/ajax/libs/photoswipe/4.1.3/default-skin/${f}"; \
    done

# Mirror the Open Sans webfont the templates request from Google Fonts. The
# stylesheet Google serves depends on the requesting User-Agent, so ask as a
# modern browser to get the woff2 variant, mirror every font file it names, and
# rewrite those URLs to the local copies.
RUN set -eu; \
    UA="Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36"; \
    mkdir -p /out/vendor/fonts/files; \
    curl -fsSL -A "$UA" -o /tmp/open-sans.css \
      "https://fonts.googleapis.com/css?family=Open+Sans:300,400,600,700"; \
    for url in $(grep -oE 'https://fonts\.gstatic\.com[^)]+\.woff2?' /tmp/open-sans.css | sort -u); do \
      curl -fsSL -o "/out/vendor/fonts/files/$(basename "$url")" "$url"; \
    done; \
    sed -E 's#https://fonts\.gstatic\.com[^)]+/([^/)]+\.woff2?)#files/\1#g' /tmp/open-sans.css \
      > /out/vendor/fonts/open-sans.css; \
    test -s /out/vendor/fonts/open-sans.css; \
    ! grep -q 'fonts\.gstatic\.com' /out/vendor/fonts/open-sans.css

# Repoint every CDN reference in the generated HTML at the mirrored copy. The
# templates use a mix of absolute and protocol-relative URLs, hence the
# optional `(https?:)?` prefix on each pattern. Galleria's theme is loaded from
# an inline Galleria.loadTheme() call rather than a <script> tag, so its URL is
# rewritten by the same pass.
RUN find /out -type f -name '*.html' -exec sed -i -E \
      -e 's#(https?:)?//code\.jquery\.com/jquery\.min\.js#/vendor/js/jquery.min.js#g' \
      -e 's#(https?:)?//code\.jquery\.com/jquery-3\.3\.1\.slim\.min\.js#/vendor/js/jquery-3.3.1.slim.min.js#g' \
      -e 's#(https?:)?//cdnjs\.cloudflare\.com/ajax/libs/popper\.js/1\.14\.0/umd/popper\.min\.js#/vendor/js/popper.min.js#g' \
      -e 's#(https?:)?//stackpath\.bootstrapcdn\.com/bootstrap/4\.1\.0/js/bootstrap\.min\.js#/vendor/js/bootstrap.min.js#g' \
      -e 's#(https?:)?//use\.fontawesome\.com/releases/v5\.1\.0/js/all\.js#/vendor/js/fontawesome-all.js#g' \
      -e 's#(https?:)?//cdn\.jsdelivr\.net/npm/galleria@1\.6\.1/dist/galleria\.min\.js#/vendor/js/galleria.min.js#g' \
      -e 's#(https?:)?//cdn\.jsdelivr\.net/npm/galleria@1\.6\.1/dist/themes/twelve/galleria\.twelve\.min\.js#/vendor/js/galleria.twelve.min.js#g' \
      -e 's#(https?:)?//cdn\.jsdelivr\.net/npm/galleria@1\.6\.1/dist/themes/twelve/galleria\.twelve\.min\.css#/vendor/css/galleria.twelve.min.css#g' \
      -e 's#(https?:)?//cdnjs\.cloudflare\.com/ajax/libs/photoswipe/4\.1\.3/photoswipe\.min\.js#/vendor/js/photoswipe.min.js#g' \
      -e 's#(https?:)?//cdnjs\.cloudflare\.com/ajax/libs/photoswipe/4\.1\.3/photoswipe-ui-default\.min\.js#/vendor/js/photoswipe-ui-default.min.js#g' \
      -e 's#(https?:)?//cdnjs\.cloudflare\.com/ajax/libs/photoswipe/4\.1\.3/photoswipe\.min\.css#/vendor/css/photoswipe.min.css#g' \
      -e 's#(https?:)?//cdnjs\.cloudflare\.com/ajax/libs/photoswipe/4\.1\.3/default-skin/default-skin\.min\.css#/vendor/photoswipe-default-skin/default-skin.min.css#g' \
      -e 's#(https?:)?//stackpath\.bootstrapcdn\.com/bootstrap/4\.1\.0/css/bootstrap\.min\.css#/vendor/css/bootstrap.min.css#g' \
      -e 's#(https?:)?//cdnjs\.cloudflare\.com/ajax/libs/font-awesome/6\.5\.2/css/all\.min\.css#/vendor/fontawesome/css/all.min.css#g' \
      -e 's#(https?:)?//fonts\.googleapis\.com/css\?family=Open\+Sans[^"]*#/vendor/fonts/open-sans.css#g' {} +

# Drop the Subresource Integrity attributes left on the tags just rewritten.
# The mirrored bytes are identical so the hashes would still match, but a stale
# integrity= on a locally-served file is a booby trap for anyone who later
# re-mirrors a patched build of the same version.
RUN find /out -type f -name '*.html' -exec sed -i -E \
      's# integrity="(sha256|sha384|sha512)-[^"]*"##g; s# crossorigin="anonymous"##g' {} + && \
    ! grep -rq 'integrity="sha' /out --include='*.html'

# Rewrite the site's own absolute self-links to root-relative paths. Templates
# and content both hardcode https://kubevirt.io/... (site.url is set to it in
# _config.yml, and many posts spell it out), which in an airgapped image points
# at nothing. Order matters: the /user-guide prefix is matched first so those
# links land on the guide mounted below, then the doubled slash that
# site.url + site.baseurl produces (https://kubevirt.io//feed.xml) is collapsed
# before the general cases -- left alone it would become the protocol-relative
# //feed.xml, which a browser reads as a request to the host "feed.xml".
RUN find /out -type f \( -name '*.html' -o -name '*.xml' \) -exec sed -i -E \
      -e 's#(https?:)?//kubevirt\.io/user-guide#/user-guide#g' \
      -e 's#https://kubevirt\.io//#/#g' \
      -e 's#https://kubevirt\.io/#/#g' \
      -e 's#https://kubevirt\.io#/#g' {} + && \
    ! grep -rq 'kubevirt\.io/user-guide' /out --include='*.html'

# Mirror every remotely-hosted <img> (today: two GitHub gist screenshots in a
# 2018 post and an asciinema cast thumbnail). The extension comes from the
# response Content-Type, not the URL, since many image URLs carry no extension
# and an SVG served as application/octet-stream will not render in an <img>.
# URLs are rewritten longest-first so one that is a prefix of another cannot
# corrupt it, and a URL that cannot be fetched is still rewritten to its local
# path, so the page degrades to a missing image rather than reaching for the
# network on load.
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
    echo "Mirrored ${n} remote image(s) in the main site"

# Assert nothing in the served markup still loads a script, stylesheet, image or
# font over the network. Two complementary checks: no <script>/<img>/<link>
# points at an absolute or protocol-relative URL (quotes optional, since a
# minifier would strip them), and none of the CDN or analytics hosts these
# templates used appears anywhere in the served HTML/CSS/JS -- the only way to
# catch a URL inside an inline script, which is where the Galleria theme loader
# lives.
#
# <iframe> is deliberately excluded: a handful of pages embed YouTube players and
# the community page embeds a Google Calendar, and neither has an offline
# equivalent -- that third-party media simply cannot be carried into an airgapped
# image. Those frames render as an unreachable box; every other asset is local.
RUN set -eu; \
    ! find /out -type f \( -name '*.html' -o -name '*.css' \) -print0 \
      | xargs -0 grep -qoE '<(script|img|link)[^>]*(src|href)=["'"'"']?(https?:)?//'; \
    ! find /out -type f \( -name '*.html' -o -name '*.css' -o -name '*.js' -o -name '*.mjs' \) \
        ! -name '*.map' -print0 \
      | xargs -0 grep -qoE '(\\?/\\?/)(cdn\.jsdelivr\.net|unpkg\.com|cdnjs\.cloudflare\.com|code\.jquery\.com|ajax\.googleapis\.com|fonts\.googleapis\.com|fonts\.gstatic\.com|[a-z]+\.bootstrapcdn\.com|use\.fontawesome\.com|kit\.fontawesome\.com|www\.googletagmanager\.com|www\.google-analytics\.com|img\.shields\.io|platform\.twitter\.com|[a-z.]*gravatar\.com|www\.redhat\.com/ma)'

RUN chmod -R 755 /out

# --- Stage 3: Serve the Offline Site ---
# Red Hat Hardened Images nginx: distroless (no shell, no package manager) and
# runs as UID 65532, so this stage listens on 8080 (see default.conf), can RUN
# nothing, and must not set a CMD -- the image's own CMD carries the flags nginx
# needs after its /usr/sbin/nginx entrypoint. See README.md for the details and
# for how to verify the signatures on the published images.
FROM registry.access.redhat.com/hi/nginx:latest

COPY --from=site-builder /out /usr/share/nginx/html
COPY --from=guide-builder /guide /usr/share/nginx/html/user-guide
COPY default.conf /etc/nginx/conf.d/default.conf

EXPOSE 8080
