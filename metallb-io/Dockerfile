# --- Stage 1: Build the Static Site ---
FROM alpine:3.20 AS builder

# HUGO_VERSION pinned to match website/../netlify.toml (metallb/metallb repo)
ARG HUGO_VERSION=0.124.1
# hugo_extended's release binary is a CGO build linked against glibc's
# libstdc++/libgcc; Alpine's musl base needs gcompat + libstdc++ + libgcc
# or it fails at runtime with "hugo: not found" (actually a dynamic-linker
# relocation failure, masked by a misleading shell error).
RUN apk add --no-cache git curl ca-certificates gcompat libstdc++ libgcc && \
    curl -sL "https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-amd64.tar.gz" \
      | tar xz -C /usr/local/bin hugo

WORKDIR /site

# The docs source lives under metallb/metallb's website/ subdirectory. The
# hugo-theme-relearn theme is vendored directly in-tree (website/themes/),
# not a git submodule, so a plain checkout is sufficient.
COPY metallb-src/website/ .

# --- OFFLINE PATCHING (pre-build) ---
# The theme (hugo-theme-relearn) already vendors Font Awesome, Mermaid+d3,
# WorkSans webfonts and a local search index, so no CDN assets are pulled in
# by default. The only external-network features it ships are opt-in:
# MathJax-via-CDN (js/mathjax/latest.js) and a Disqus comments partial.
# Neither is enabled by this site's config.toml/content, but strip the
# Disqus template call defensively so it can never render a request even if
# a future content change sets a shortname.
RUN sed -i -E '/_internal\/disqus\.html/d' themes/hugo-theme-relearn/layouts/partials/custom-comments.html

# Build the site. baseURL "/" comes from config.toml; output to /out.
RUN hugo --minify --destination /out --baseURL / && \
    test -f /out/index.html || (echo "FATAL: no content built" && exit 1)

# --- OFFLINE PATCHING (post-build) ---
# Belt-and-braces: strip any Google Fonts / analytics tags that might have
# slipped in, and neutralize the (unused-by-default) MathJax CDN loader
# script in case a page ever sets math=true.
RUN find /out -type f -name "*.html" -exec sed -i -E 's#<link[^>]*fonts\.(googleapis|gstatic)\.com[^>]*>##g' {} + && \
    find /out -type f -name "*.html" -exec sed -i -E '/_internal\/(google_analytics|disqus)/d' {} +
RUN rm -f /out/js/mathjax/latest.js /out/js/mathjax/sre/sre_browser.js

# The theme's footer (menu-footer.html) hardcodes a "Deploys by Netlify"
# badge: an <a href="https://www.netlify.com"><img src="https://www.netlify.com/
# img/global/badges/netlify-color-accent.svg"></a> block, present on every
# rendered page regardless of where the site is actually hosted. That <img>
# fires a real request (and a redirect to
# netlify.com/assets/badges/netlify-badge-color-accent.svg) on load, which is
# unreachable in an airgapped image. Strip the whole anchor+image block
# (hugo --minify collapses it onto a single line, so plain sed suffices) and
# the empty <p></p> it leaves behind.
RUN find /out -type f -name "*.html" -exec sed -i -E \
      's#<a[^>]*href="?https?://(www\.)?netlify\.com/?"?[^>]*>\s*<img[^>]*netlify\.com[^>]*>\s*</a>##g; s#<p>\s*</p>##g' {} +

RUN chmod -R 755 /out

# --- Stage 2: Serve the Offline Site ---
FROM nginx:alpine

COPY --from=builder /out /usr/share/nginx/html
COPY default.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
