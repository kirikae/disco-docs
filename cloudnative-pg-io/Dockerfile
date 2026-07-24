FROM node:22-alpine AS builder
WORKDIR /app

# The upstream docs source (github.com/cloudnative-pg/docs-new) is a
# Docusaurus 3 site. Only the "website" subdirectory is needed to build it.
COPY cnpg-src/website ./website
WORKDIR /app/website

# --- OFFLINE PATCHING (pre-build) ---
# Upstream serves this site mounted under https://cloudnative-pg.io/docs/, so
# baseUrl is "/docs/". This container serves the docs standalone at the
# domain root, so repoint baseUrl to "/" (all internal links, the version
# dropdown, and the plugin-client-redirects root->latest-version redirect are
# derived from this and follow automatically).
RUN sed -i "s#baseUrl: '/docs/'#baseUrl: '/'#" docusaurus.config.ts

# Docs search is already fully local (@easyops-cn/docusaurus-search-local,
# prebuilt index, no Algolia/API calls) and no analytics/Google Fonts are
# configured anywhere in this repo (verified against docusaurus.config.ts,
# src/css/custom.css and every markdown doc).
#
# Two external asset references were found in the doc content itself (present
# in "docs/" *and* duplicated into every "versioned_docs/version-*/" copy):
#
# 1. faq.md embeds a live, server-rendered GitHub star-history chart
#    (api.star-history.com) - a dynamic image that cannot be meaningfully
#    vendored offline, so it is dropped.
RUN find docs versioned_docs -name "faq.md" -exec sed -i '/api\.star-history\.com/d' {} +

# 2. index.md hotlinks the CNCF logo from github.com - vendor it locally
#    instead so the landing page image doesn't require a live fetch.
RUN mkdir -p static/img && \
    wget -qO static/img/cncf-color.png \
      "https://github.com/cncf/artwork/blob/main/other/cncf/horizontal/color/cncf-color.png?raw=true" && \
    find docs versioned_docs -name "index.md" -exec sed -i \
      's#https://github\.com/cncf/artwork/blob/main/other/cncf/horizontal/color/cncf-color\.png?raw=true#/img/cncf-color.png#g' {} +

RUN corepack enable && yarn install --frozen-lockfile
RUN yarn build
RUN test -f build/index.html || (echo "FATAL: no content" && exit 1)

# --- OFFLINE PATCHING (post-build) ---
# Defensive belt-and-braces sweep for Google Fonts links, in case a future
# upstream change (or a theme default) introduces one.
RUN find build -type f -name "*.html" -exec sed -i -E 's#<link[^>]*fonts\.(googleapis|gstatic)\.com[^>]*>##g' {} +

RUN chmod -R 755 build

FROM nginx:alpine
COPY --from=builder /app/website/build /usr/share/nginx/html
COPY default.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
