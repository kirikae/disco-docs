# --- Stage 1: Build the Static Site ---
# Node 24: upstream stackabletech/documentation now requires Node >= 24 (its
# .nvmrc pins 24 and package.json declares engines.node ">= 24.0.0"). The UI
# bundler (rolldown) imports `styleText` from node:util, which only exists on
# Node 20.12+, so the previous Node 18 pin fails the UI bundle step of
# `make build-prod`. Antora 3.1.x and Pagefind both run fine on 24.
FROM node:24-bookworm-slim AS builder

WORKDIR /app

# Prerequisites for the Antora/gulp build and git-based content fetching.
RUN apt-get update && apt-get install -y \
    git \
    make \
    python3 \
    g++ \
    curl \
    && rm -rf /var/lib/apt/lists/*

RUN git config --global --add safe.directory '*'

# Clone the FULL documentation repo (all branches, full history). The production
# playbook (antora-playbook.yml) builds every release as a separate version by
# reading them from the `url: .` content source, so the whole branch topology
# must be present. A shallow, single-branch CI checkout (the pattern used by the
# other sites here) would not work, hence we clone in-image instead of COPY.
RUN git clone https://github.com/stackabletech/documentation.git .

# The documentation-ui lives in a git submodule; it must be populated before
# `npm ci` so the npm workspace ("ui") resolves.
RUN git submodule update --init --recursive

# Materialise a local branch for every remote release branch. Antora's `url: .`
# multi-version source resolves against local refs; this is a tolerant,
# non-destructive equivalent of the upstream `netlify-fetch` checkout loop
# (which resets the worktree and aborts on the first bad ref).
RUN for ref in $(git for-each-ref --format='%(refname:short)' refs/remotes/origin | grep -E '^origin/release[/-]'); do \
      git branch --force "${ref#origin/}" "$ref" || true; \
    done

# --- OFFLINE PATCHING & CONFIG OVERRIDES ---
# 1. The prod playbook emits the "stable" version segment as Netlify-style
#    redirects (`redirect_facility: netlify`), which a plain nginx cannot serve.
#    Switch to `static` so Antora writes real static HTML redirect pages.
# 2. Disable UI usage tracking so no external analytics script is injected.
# 3. The mermaid extension injects `<script type="module">import mermaid from
#    '<url>'</script>` into every page, defaulting to a jsDelivr CDN. Point it at
#    a locally-vendored copy (added to the built site below) so diagrams render
#    offline.
RUN sed -i -E "s/redirect_facility:\s*netlify/redirect_facility: static/g" antora-playbook.yml && \
    sed -i -E "s/enable_tracking:\s*true/enable_tracking: false/g" antora-playbook.yml && \
    sed -i -E "s#mermaid_library_url:.*#mermaid_library_url: /vendor/mermaid/mermaid.esm.min.mjs#g" antora-playbook.yml

# Install dependencies (root + the "ui" workspace).
RUN npm ci || npm install

# Full production build: builds the UI bundle, then fetches every operator/demo
# repo from GitHub and compiles the multi-version site into build/site.
RUN make build-prod

# Vendor Mermaid so diagrams render offline (see the playbook URL patch above).
# The whole dist/ is copied because its chunks are loaded via relative imports;
# the absolute `/vendor/mermaid/` URL then resolves them from any page depth.
RUN npm install --no-save --prefix /opt/mermaid mermaid@10 && \
    mkdir -p build/site/vendor/mermaid && \
    cp -a /opt/mermaid/node_modules/mermaid/dist/. build/site/vendor/mermaid/

# Build the offline (fully static) Pagefind search index over the generated site.
RUN make build-search-index

# Strip any external font/analytics references that slipped through, for offline use.
RUN find build/site -type f -name "*.html" -exec sed -i -E 's#<link[^>]*fonts\.(googleapis|gstatic)\.com[^>]*>##g' {} + || true && \
    find build/site -type f -name "*.html" -exec sed -i -E 's#<script[^>]*google-analytics[^>]*>.*</script>##g' {} + || true

# --- Stage 2: Serve the Offline Site ---
FROM nginx:alpine

COPY --from=builder /app/build/site /usr/share/nginx/html
COPY default.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
