# --- Stage 1: Build the Static Site ---
FROM node:24-bookworm-slim AS builder

WORKDIR /app

# 1. Install prerequisites required for native Node modules and git-based tooling
RUN apt-get update && apt-get install -y \
    git \
    curl \
    python3 \
    make \
    g++ \
    && rm -rf /var/lib/apt/lists/*

RUN git config --global --add safe.directory '*'

# The docs repo pins pnpm via the "packageManager" field in package.json;
# corepack activates the exact version it expects.
RUN corepack enable

# Copy the Gitea docs source (hosted on gitea.com, not GitHub)
COPY gitea-docs/ .

# 2. Fetch the "Awesome Gitea" list content that gets embedded into the docs
# (mirrors the "prepare awesome list" step in the upstream .gitea CI workflow)
RUN make prepare-awesome-latest prepare-awesome\#25 prepare-awesome\#24 prepare-awesome\#23 prepare-awesome\#22 || true

# 3. Install dependencies
RUN pnpm install --frozen-lockfile || pnpm install

# 4. --- OFFLINE PATCHING & CONFIG OVERRIDES ---
# Defensively downgrade strict broken-link errors to warnings so external typos
# don't crash the build (upstream already defaults to "warn", but this guards
# against that changing).
RUN sed -i -E "s/onBrokenLinks: ['\"]throw['\"]/onBrokenLinks: 'warn'/g" docusaurus.config.js || true

# 5. Build the Docusaurus site (English only, to save space and build time;
# all versions configured in docusaurus.config.js are compiled natively into
# /app/build). Upstream's own Makefile sets a raised heap size for the same
# reason the multi-version/multi-plugin build otherwise OOMs.
ENV NODE_OPTIONS="--max-old-space-size=8192"
RUN pnpm exec docusaurus build --locale en-us

# 6. Strip External Dependencies (Offline Mode)
# Removes Google Fonts links, Google Analytics/Tag Manager, and the Plausible
# analytics script that upstream loads via docusaurus-plugin-plausible.
#
# Docusaurus emits the whole <head> as a single minified line, so the script
# body must be matched with `[^<]*` (stops at the next tag) rather than a
# greedy `.*` — a greedy match spans past these (empty-bodied) tracker
# scripts all the way to the LAST `</script>` on the line, which deletes the
# real stylesheet <link> and the runtime/main bundle <script> tags that
# follow later on the same line and breaks the site's styling/hydration.
RUN find build -type f -name "*.html" -exec sed -i -E 's/<link[^>]*fonts.(googleapis|gstatic).com[^>]*>//g' {} + || true && \
    find build -type f -name "*.html" -exec sed -i -E 's/<link[^>]*preconnect[^>]*(google-analytics|googletagmanager|plausible)\.[^>]*>//g' {} + || true && \
    find build -type f -name "*.html" -exec sed -i -E 's/<script[^>]*google-analytics[^>]*>[^<]*<\/script>//g' {} + || true && \
    find build -type f -name "*.html" -exec sed -i -E 's/<script[^>]*googletagmanager[^>]*>[^<]*<\/script>//g' {} + || true && \
    find build -type f -name "*.html" -exec sed -i -E 's/<script[^>]*plausible\.io[^>]*>[^<]*<\/script>//g' {} + || true

# --- Stage 2: Serve the Offline Site ---
FROM nginx:alpine

COPY --from=builder /app/build /usr/share/nginx/html
COPY default.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
