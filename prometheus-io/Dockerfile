# --- Stage 1: Build the Static Site ---
# The current prometheus.io/docs is a Next.js 15 + React 19 app (a rewrite of
# the previously Hugo-based site) that statically exports to `out/` via
# `output: "export"` in next.config.ts. Node 22 comfortably satisfies Next
# 15.5's Node >=18.18 requirement and matches the version already used by the
# helm-sh site in this repo.
FROM node:22-bookworm-slim AS builder

WORKDIR /app

# git is required because the pre-build scripts (scripts/fetch-repo-docs.ts)
# clone the Prometheus and Alertmanager repos (bare clone + per-version git
# worktrees) to pull in their versioned docs/ directories at build time.
RUN apt-get update && apt-get install -y \
    git \
    && rm -rf /var/lib/apt/lists/*

RUN git config --global --add safe.directory '*'

# Copy the Prometheus docs website source (checked out by CI, see the
# build-prometheus-docs job in build-offline-docs.yml).
COPY prometheus-docs/ .

# Optional: a GitHub token to raise the GitHub API rate limit (60 req/hour
# unauthenticated vs. 5000 req/hour authenticated) while
# fetch-repo-docs.ts/fetch-downloads-info.ts fetch release and version
# metadata for the Prometheus, Alertmanager, and exporter repos. CI supplies
# this via a BuildKit secret (not a build-arg/ENV) so the token is never
# written into an image layer or the pushed layer-cache history.
RUN --mount=type=secret,id=GITHUB_TOKEN \
    if [ -s /run/secrets/GITHUB_TOKEN ]; then \
      echo "GITHUB_TOKEN=$(cat /run/secrets/GITHUB_TOKEN)" > .env; \
    fi

# 1. Install dependencies
RUN npm ci || npm install

# 2. --- OFFLINE PATCHING ---
# Disable the Kapa AI search/chat widget: KapaWidget.tsx injects a
# <script src="https://widget.kapa.ai/kapa-widget.bundle.js"> at runtime
# whenever docsConfig.kapa is set, which it is by default. The component
# already no-ops via `if (!docsConfig.kapa) { return null; }`, so force that
# branch unconditionally. (We can't just delete the `kapa: {...}` block from
# docs-config.ts instead -- `kapa` is a required, non-optional field of the
# DocsConfig type, so `docsConfig satisfies DocsConfig` fails to type-check
# without it.)
#
# No patch is needed for Google Analytics: layout.tsx only renders
# <GoogleAnalytics> when NEXT_PUBLIC_GOOGLE_ANALYTICS_ID is set, and we never
# set it, so it's absent by default.
#
# No patch is needed for Google Fonts (Inter/Lato): they're loaded via
# next/font/google, which downloads and self-hosts the font files at build
# time. The built site serves them from its own origin, with no runtime
# request to fonts.googleapis.com/fonts.gstatic.com.
RUN sed -i -E 's/if \(!docsConfig\.kapa\) \{/if (true) {/' src/components/KapaWidget.tsx

# 3. Full production build: fetches versioned docs from the Prometheus and
# Alertmanager repos, fetches download/release metadata for the downloads
# page, builds the Next.js static export to out/, and builds the offline
# Pagefind search index over it (via the postbuild script).
RUN npm run build-all

# --- Stage 2: Serve the Offline Site ---
FROM nginx:alpine

COPY --from=builder /app/out /usr/share/nginx/html
COPY default.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
