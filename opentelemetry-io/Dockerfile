# --- Stage 1: Build the Static Site ---
# opentelemetry.io is a Hugo (Docsy theme) site with a heavy npm-driven build
# pipeline and git-submodule content (spec, semantic-conventions, community,
# etc. - see .gitmodules). This image bundles Hugo Extended + Node + git on
# top of Debian, which (unlike the Alpine variant) ships bash and a full perl
# out of the box - both are required by the project's own build scripts
# (npm's "seq" helper shells out to bash, and scripts/update-semconv-mounts.pl
# needs a non-minimal perl).
FROM hugomods/hugo:debian-node-git-0.164.0 AS builder

USER root
WORKDIR /app
RUN git config --global --add safe.directory '*'

# Copy the opentelemetry.io website source (checked out with
# `submodules: true` in CI, so content-modules/* and themes/docsy are already
# populated at their pinned commits - see .gitmodules)
COPY opentelemetry-website/ .

# 1. --- OFFLINE PATCHING ---
# Vendor jQuery, which the Docsy theme loads from code.jquery.com. The
# fetched file is byte-identical to what the CDN serves (same pinned
# version), so the existing SRI hash still matches after repointing the src.
RUN mkdir -p static/js && \
    for jq in $(grep -rhoE 'https://code\.jquery\.com/jquery-[0-9.]+\.min\.js' themes/docsy/theme/layouts | sort -u); do \
      wget -qO "static/js/$(basename "$jq")" "$jq"; \
    done && \
    find themes/docsy/theme/layouts -type f -name "*.html" \
      -exec sed -i -E 's#https://code\.jquery\.com/(jquery-[0-9.]+\.min\.js)#/js/\1#g' {} +

# Remove the Kapa.ai AI-chat widget: a third-party script plus a hosted MCP
# endpoint, with no offline equivalent to vendor. Google Fonts and Google
# Analytics/Tag Manager do not need patching - they're already disabled
# upstream ($td-enable-google-fonts: false in the Docsy theme defaults, and
# no gtag/GTM id is configured anywhere in this site).
RUN perl -0777 -pi -e 's#<script\s+async\s+src="https://widget\.kapa\.ai/kapa-widget\.bundle\.js".*?</script>##s' \
      layouts/_partials/hooks/head-end.html

# Strip any remaining SRI/crossorigin attributes so patched or locally
# fingerprinted assets can never fail an integrity check offline.
RUN find . -path './node_modules' -prune -o -path './.git' -prune -o \
      -type f \( -name "*.html" -o -name "*.js" \) -print0 \
      | xargs -0 sed -i -e 's/ integrity="sha[0-9]*-[^"]*"//g' -e 's/ crossorigin="anonymous"//g'

# 2. Install npm dependencies. This also runs the project's own "prepare"
# lifecycle script (git submodule update --init + the Docsy theme's own
# postinstall), which is a no-op for the submodules since they're already
# checked out at the pinned commits above.
RUN npm install

# 3. Build the production site. npm's pre-script hook auto-runs
# "prebuild:production" first, which pulls in the spec/semconv/community
# content from the submodules (scripts/content-modules/cp-pages.sh) and
# generates the collector config-schema types before Hugo renders the site.
RUN npm run build:production

# --- Stage 2: Serve the Offline Site ---
FROM nginx:alpine

COPY --from=builder /app/public /usr/share/nginx/html
COPY default.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
