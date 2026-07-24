# --- Stage 1: Build the Static Site ---
# thanos.io is a Hugo site (no theme, layouts live in-repo under website/layouts).
# Docs content is assembled from docs/ into website/docs-pre-processed by the
# `mdox` tool (github.com/bwplotka/mdox), a pinned pseudo-version built via the
# repo's bingo tooling (.bingo/mdox.mod). Hugo version (0.101.0, extended) is
# pinned in both netlify.toml and .bingo/Variables.mk.
FROM golang:1.23-alpine AS builder

# gcompat/libstdc++ let the glibc-linked, upstream-released Hugo binary run on
# this musl-based Alpine image.
RUN apk add --no-cache git bash make curl ca-certificates gcompat libstdc++

ARG HUGO_VERSION=0.101.0
RUN curl -sL "https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_Linux-64bit.tar.gz" \
    | tar xz -C /usr/local/bin hugo

WORKDIR /src
COPY thanos-src/ .
RUN git config --global --add safe.directory '*'

# 1. Build mdox, pinned via the repo's own bingo module file. This mirrors what
#    `make web-pre-process` does automatically through .bingo/Variables.mk.
RUN cd .bingo && go build -mod=mod -modfile=mdox.mod -o /usr/local/bin/mdox github.com/bwplotka/mdox

# --- OFFLINE PATCHING (pre-build, markdown sources) ---
# A handful of docs/ pages embed architecture diagrams hosted on Google Drawings
# (docs.google.com/drawings/.../pub) as <img> sources. Vendor those in so no
# runtime request escapes the container.
RUN mkdir -p website/static/vendor/images && \
    curl -sSL -o website/static/vendor/images/receive-split-before.png "https://docs.google.com/drawings/d/e/2PACX-1vTfko27YB_3ab7ZL8ODNG5uCcrpqKxhmqaz3lW-yhGN3_oNxkTrqXmwwlcZjaWf3cGgAJIM4CMwwkEV/pub?w=960&h=720" && \
    curl -sSL -o website/static/vendor/images/receive-split-after.png "https://docs.google.com/drawings/d/e/2PACX-1vTVrtCGjR4iMbrU7Kj6QAn1a1m4fr-kvoQVDAK4lzQ_wWfXfpLLEE9HB948-WHI5ZG6s1iGWt51R593/pub?w=960&h=720" && \
    curl -sSL -o website/static/vendor/images/read-write-bucket-diagram-1.png "https://docs.google.com/drawings/d/e/2PACX-1vSZjddrWBkf2ynczeIMSzGo-yqe7Cg9X-TG_LjyWAP_ToyZolMEG_UnbNlvoSetRhDLmlfABd_rJ4W9/pub?w=960&h=720" && \
    curl -sSL -o website/static/vendor/images/read-write-bucket-diagram-2.png "https://docs.google.com/drawings/d/e/2PACX-1vTVX4WgIa4O0rcvN8R_oZJBDOGNe0-eTJW7Ucqw0CnwcjuswATupMRD3r97b94cW6xasVsU7MPQPpVf/pub?w=950&h=720" && \
    curl -sSL -o website/static/vendor/images/quick-tutorial-sidecar.png "https://docs.google.com/drawings/d/e/2PACX-1vSJd32gPh8-MC5Ko0-P-v1KQ0Xnxa0qmsVXowtkwVGlczGfVW-Vd415Y6F129zvh3y0vHLBZcJeZEoz/pub?w=960&h=720" && \
    curl -sSL -o website/static/vendor/images/quick-tutorial-receive.png "https://docs.google.com/drawings/d/e/2PACX-1vRdYP__uDuygGR5ym1dxBzU6LEx5v7Rs1cAUKPsl5BZrRGVl5YIj5lsD_FOljeIVOGWatdAI9pazbCP/pub?w=960&h=720" && \
    sed -i \
      -e 's#https://docs\.google\.com/drawings/d/e/2PACX-1vTfko27YB_3ab7ZL8ODNG5uCcrpqKxhmqaz3lW-yhGN3_oNxkTrqXmwwlcZjaWf3cGgAJIM4CMwwkEV/pub?w=960&h=720#/vendor/images/receive-split-before.png#' \
      -e 's#https://docs\.google\.com/drawings/d/e/2PACX-1vTVrtCGjR4iMbrU7Kj6QAn1a1m4fr-kvoQVDAK4lzQ_wWfXfpLLEE9HB948-WHI5ZG6s1iGWt51R593/pub?w=960&h=720#/vendor/images/receive-split-after.png#' \
      docs/proposals-accepted/202012-receive-split.md && \
    sed -i \
      -e 's#https://docs\.google\.com/drawings/d/e/2PACX-1vSZjddrWBkf2ynczeIMSzGo-yqe7Cg9X-TG_LjyWAP_ToyZolMEG_UnbNlvoSetRhDLmlfABd_rJ4W9/pub?w=960&h=720#/vendor/images/read-write-bucket-diagram-1.png#' \
      -e 's#https://docs\.google\.com/drawings/d/e/2PACX-1vTVX4WgIa4O0rcvN8R_oZJBDOGNe0-eTJW7Ucqw0CnwcjuswATupMRD3r97b94cW6xasVsU7MPQPpVf/pub?w=950&amp;h=720#/vendor/images/read-write-bucket-diagram-2.png#' \
      docs/proposals-done/201901-read-write-operations-bucket.md && \
    sed -i \
      -e 's#https://docs\.google\.com/drawings/d/e/2PACX-1vSJd32gPh8-MC5Ko0-P-v1KQ0Xnxa0qmsVXowtkwVGlczGfVW-Vd415Y6F129zvh3y0vHLBZcJeZEoz/pub?w=960&h=720#/vendor/images/quick-tutorial-sidecar.png#' \
      -e 's#https://docs\.google\.com/drawings/d/e/2PACX-1vRdYP__uDuygGR5ym1dxBzU6LEx5v7Rs1cAUKPsl5BZrRGVl5YIj5lsD_FOljeIVOGWatdAI9pazbCP/pub?w=960&h=720#/vendor/images/quick-tutorial-receive.png#' \
      docs/quick-tutorial.md

# 2. Assemble docs into the Hugo site. Upstream's full `scripts/website/websitepreprocess.sh`
#    also builds one docs tree per historical release branch (v0.10 .. v0.41+) by
#    fetching the entire branch history from GitHub; we intentionally build only the
#    default/current ("tip") docs tree, mirroring what that script does for HEAD.
RUN rm -rf website/docs-pre-processed && \
    OUTPUT_CONTENT_DIR="website/docs-pre-processed" && \
    INPUT_DIR="docs" OUTPUT_DIR="${OUTPUT_CONTENT_DIR}/tip" EXTERNAL_GLOB_REL="../" \
      mdox transform --log.level=info --config-file=.mdox.yaml && \
    bash scripts/website/mdoxpostprocess.sh "${OUTPUT_CONTENT_DIR}/tip" 100000

WORKDIR /src/website

# 3. --- OFFLINE VENDORING: fetch the pinned CDN assets referenced by the theme ---
RUN mkdir -p static/vendor/bootstrap static/vendor/fontawesome/webfonts static/vendor/js && \
    curl -sSL -o static/vendor/bootstrap/bootstrap.min.css https://stackpath.bootstrapcdn.com/bootstrap/4.2.1/css/bootstrap.min.css && \
    curl -sSL -o static/vendor/js/bootstrap.min.js https://stackpath.bootstrapcdn.com/bootstrap/4.5.0/js/bootstrap.min.js && \
    curl -sSL -o static/vendor/js/jquery-3.5.1.slim.min.js https://code.jquery.com/jquery-3.5.1.slim.min.js && \
    curl -sSL -o static/vendor/js/popper.min.js https://cdn.jsdelivr.net/npm/popper.js@1.16.0/dist/umd/popper.min.js && \
    curl -sSL -o static/vendor/fontawesome/all.css https://use.fontawesome.com/releases/v5.7.1/css/all.css && \
    for f in fa-brands-400 fa-regular-400 fa-solid-900; do \
      for ext in woff2 woff ttf; do \
        curl -sSL -o static/vendor/fontawesome/webfonts/$f.$ext https://use.fontawesome.com/releases/v5.7.1/webfonts/$f.$ext; \
      done; \
    done && \
    # FA's all.css references fonts via relative `../webfonts/`, which from
    # /vendor/fontawesome/all.css resolves to /vendor/webfonts/ (404). Point it
    # at the actual vendored location so the icon webfonts load offline.
    sed -i 's#\.\./webfonts/#/vendor/fontawesome/webfonts/#g' static/vendor/fontawesome/all.css

# 4. --- OFFLINE PATCHING (post-fetch, layouts) ---
#    Point CDN references at the vendored copies, strip SRI/crossorigin (no longer
#    valid once the bytes are served locally), and drop the two things that can't
#    be vendored because they call a live external API at runtime: the Algolia
#    docsearch widget and the Netlify footer badge. Google Analytics is only
#    injected by baseof.html when HUGO_ENV=production, which we never set below.
RUN sed -i \
      -e 's#https://stackpath\.bootstrapcdn\.com/bootstrap/4\.2\.1/css/bootstrap\.min\.css#/vendor/bootstrap/bootstrap.min.css#' \
      -e 's#https://use\.fontawesome\.com/releases/v5\.7\.1/css/all\.css#/vendor/fontawesome/all.css#' \
      -e 's#https://code\.jquery\.com/jquery-3\.5\.1\.slim\.min\.js#/vendor/js/jquery-3.5.1.slim.min.js#' \
      -e 's#https://cdn\.jsdelivr\.net/npm/popper\.js@1\.16\.0/dist/umd/popper\.min\.js#/vendor/js/popper.min.js#' \
      -e 's#https://stackpath\.bootstrapcdn\.com/bootstrap/4\.5\.0/js/bootstrap\.min\.js#/vendor/js/bootstrap.min.js#' \
      -e '/cdn\.jsdelivr\.net\/npm\/docsearch\.js/d' \
      -e 's/ integrity="sha[0-9]*-[^"]*"//g' -e 's/ crossorigin="anonymous"//g' \
      layouts/_default/baseof.html && \
    sed -i -e '/https:\/\/www\.netlify\.com/d' layouts/_default/baseof.html && \
    : > layouts/partials/searchConfig/index.html && \
    : > layouts/partials/searchConfig/docs.html && \
    find . -type f -name "*.html" -exec sed -i -E 's#<link[^>]*fonts\.(googleapis|gstatic)\.com[^>]*>##g' {} +

# 5. Build the site (HUGO_ENV left unset so the google_analytics_async partial is
#    never rendered) and assert we produced real content.
RUN hugo --config hugo.yaml --minify -b "/" -d /out && test -f /out/index.html || (echo "FATAL: no content" && exit 1)
RUN chmod -R 755 /out

# --- Stage 2: Serve the Offline Site ---
FROM nginx:alpine

COPY --from=builder /out /usr/share/nginx/html
COPY default.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
