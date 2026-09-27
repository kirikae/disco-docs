# --- Stage 1: Build the Static Site ---
# konpyutaika.github.io/nifikop is a Docusaurus 3 site living in
# konpyutaika/nifikop under site/, as a yarn workspace whose single member is
# site/website. Its current docs are site/docs (which is what the config's
# docs.path "../docs" resolves to, relative to site/website -- not the docs/
# directory at the repo root, which is a separate dev/tutorial tree), plus 35
# archived versions under site/website/versioned_docs.
FROM docker.io/library/node:22-bookworm-slim AS builder

WORKDIR /app

# curl: vendoring the one CDN script the site injects. git: not needed to build,
# but Docusaurus shells out to it while resolving doc metadata, and a clear
# "not a git repository" is easier to read than a missing binary.
RUN apt-get update && apt-get install -y \
    git \
    curl \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

RUN git config --global --add safe.directory '*'

# Only site/ is needed; the workspace root is site/package.json.
COPY nifikop-src/site/ .

RUN yarn install --frozen-lockfile

# --- OFFLINE PATCHING (pre-build) ---
# 1. Serve from the container root instead of the /nifikop/ path GitHub Pages
#    publishes the project site under. Upstream's own netlify.toml does exactly
#    this rewrite for deploy previews, so it is a supported configuration rather
#    than something invented here; every internal link, the version dropdown and
#    the asset URLs are derived from baseUrl and follow automatically. Both forms
#    are handled because netlify.toml patches a `const baseUrl` line that only
#    exists on some branches.
RUN sed -i -e "s|const baseUrl .*|const baseUrl = '/'|g" \
           -e "s#baseUrl: '/nifikop/'#baseUrl: '/'#" website/docusaurus.config.js && \
    grep -qE "baseUrl: '/'" website/docusaurus.config.js

# 2. Two blog posts link into the docs with the /nifikop/ prefix hardcoded in
#    the markdown rather than going through Docusaurus. Those are the only links
#    step 1 cannot fix for itself, and they 404 once the prefix is gone.
#    Absolute links to the live site elsewhere in the old (2020) posts are left
#    alone: several point at a docs layout that no longer exists, so rewriting
#    them would turn a link that works for an online reader into a local 404.
RUN find docs website/versioned_docs website/blog -type f \( -name '*.md' -o -name '*.mdx' \) \
      -exec sed -i 's#](/nifikop/#](/#g' {} + && \
    ! grep -rq '](/nifikop/' docs website/versioned_docs website/blog

# 3. Drop the Expo Snack embed script. It is inherited from the
#    react-native-website template this site was forked from, loads on all ~3300
#    pages, and nothing in nifikop's content uses a SnackPlayer block (verified
#    across docs/, versioned_docs/ and blog/). The snackPlayerInitializer client
#    module can stay: every call it makes is guarded behind `window.ExpoSnack &&`,
#    so with the script gone it simply does nothing.
RUN sed -i '/snack\.expo\.io\/embed\.js/d' website/docusaurus.config.js && \
    ! grep -q 'snack\.expo\.io' website/docusaurus.config.js

# 4. Vendor the focus-visible polyfill, the other entry in the site's `scripts`
#    array, which is otherwise fetched from jsDelivr on every page. The URL is
#    read out of the config rather than hardcoded here, so a version bump
#    upstream cannot leave us serving a different build than the site asked for.
RUN set -eu; \
    cfg=website/docusaurus.config.js; \
    url=$(grep -oE 'https://cdn\.jsdelivr\.net/npm/focus-visible@[0-9.]+/dist/focus-visible\.min\.js' "$cfg" | head -1); \
    test -n "$url"; \
    mkdir -p website/static/vendor; \
    curl -fsSL -o website/static/vendor/focus-visible.min.js "$url"; \
    sed -i "s#'${url}'#'/vendor/focus-visible.min.js'#" "$cfg"; \
    ! grep -q 'cdn\.jsdelivr\.net' "$cfg"; \
    grep -q "'/vendor/focus-visible.min.js'" "$cfg"

# 5. Replace Algolia DocSearch with a local index.
#
#    DocSearch is a hosted service: the search box queries algolia.net at run
#    time, and the theme also emits a preconnect to the Algolia DSN on every
#    page. Neither works offline, so the config block has to go regardless.
#
#    Simply deleting it would leave 35 versions of documentation with no search
#    at all, which in a disconnected environment is where search matters most --
#    there is no web to fall back on. So the block is swapped for
#    @easyops-cn/docusaurus-search-local, which builds a lunr index at build time
#    and ships it as a static file per docs version. This is an addition to
#    upstream's configuration, not a patch of it; it is the same plugin other
#    projects mirrored in this repo already use upstream.
RUN sed -i '/^    algolia: {$/,/^    },$/d' website/docusaurus.config.js && \
    ! grep -q 'algolia:' website/docusaurus.config.js
#    Pinned, like every other tool version in this repo, so a plugin release
#    cannot change the shipped search behaviour between two nightly builds.
ARG SEARCH_PLUGIN_VERSION=0.55.3
RUN yarn --cwd website add "@easyops-cn/docusaurus-search-local@${SEARCH_PLUGIN_VERSION}"
RUN sed -i "s#plugins: \['@docusaurus/theme-live-codeblock', 'docusaurus-plugin-sass', './sitePlugin'\],#plugins: ['@docusaurus/theme-live-codeblock', 'docusaurus-plugin-sass', './sitePlugin', [require.resolve('@easyops-cn/docusaurus-search-local'), {hashed: true, indexBlog: true, docsRouteBasePath: '/docs'}]],#" \
      website/docusaurus.config.js && \
    grep -q 'easyops-cn/docusaurus-search-local' website/docusaurus.config.js

# 6. Remove the GitHub star-count badge from the landing page. It is an <iframe>
#    served by ghbtns.com whose whole purpose is reporting a live star count, so
#    there is nothing to vendor -- offline it renders as an unreachable box.
#    Only the wrapper <span> and the <iframe> go; the "Get Started" button beside
#    it is untouched.
RUN sed -i '/<span className={styles.indexCtasGitHubButtonWrapper}>/,/<\/span>/d' \
      website/src/pages/index.js && \
    ! grep -q 'ghbtns\.com' website/src/pages/index.js && \
    grep -q 'indexCtasGetStartedButton' website/src/pages/index.js

# 7. Docusaurus resolves showLastUpdateAuthor/showLastUpdateTime by running
#    `git log` per doc, which aborts the build outright when there is no
#    repository (only site/ is copied in) and, on the depth-1 checkout CI makes,
#    would otherwise stamp every one of ~3300 pages with the same single commit's
#    date and author. There is no fallback setting, so turn both off rather than
#    publish a date that is wrong everywhere.
RUN sed -i -E "s/showLastUpdate(Author|Time): true/showLastUpdate\1: false/g" \
      website/docusaurus.config.js && \
    ! grep -qE 'showLastUpdate(Author|Time): true' website/docusaurus.config.js

# 8. Downgrade broken-link errors to warnings, as the other Docusaurus sites here
#    do: an upstream typo in a link should not take the nightly build down.
RUN sed -i -E "s/onBrokenLinks: ['\"]throw['\"]/onBrokenLinks: 'warn'/" website/docusaurus.config.js

# The build compiles the current docs plus all 35 archived versions in one pass,
# which needs more heap than Node's default -- the same bump upstream's
# netlify.toml applies.
RUN cd website && NODE_OPTIONS=--max_old_space_size=4096 yarn build && \
    test -f build/index.html || (echo "FATAL: Docusaurus generated no content" && exit 1)

# Assert the local search index really was produced, for the current docs and
# for the archived versions. Without this, a plugin that silently stopped
# emitting an index would leave a search box that returns nothing.
RUN test -s website/build/search-index.json && \
    test -s website/build/docs/next/search-index.json && \
    test "$(find website/build/docs -name search-index.json | wc -l)" -gt 30

# --- OFFLINE PATCHING (post-build) ---
# Mirror every remotely-hosted <img>. Today that is the blog author's avatar from
# avatars0.githubusercontent.com, referenced from blog/authors.yml and rendered
# onto all 66 blog pages. The extension comes from the response Content-Type, not
# the URL, since avatar URLs carry query strings and no extension; URLs are
# rewritten longest-first so one that is a prefix of another cannot corrupt it;
# and a URL that cannot be fetched is still rewritten to its local path, so the
# page degrades to a missing image rather than reaching for the network on load.
RUN set -eu; \
    mkdir -p website/build/vendor/img; \
    find website/build -type f -name '*.html' -print0 \
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
      name="${n}.${ext}"; \
      cp /tmp/mirror.bin "website/build/vendor/img/${name}" 2>/dev/null || true; \
      find website/build -type f -name '*.html' -exec sed -i "s#${url}#/vendor/img/${name}#g" {} +; \
    done < /tmp/ext-img-urls.txt; \
    echo "Mirrored ${n} remote image(s)"

# Belt-and-braces sweep for font/analytics markup. None is configured anywhere in
# this site (no Google Fonts import in src/css, no gtag/GA plugin), but a future
# upstream change could add one.
RUN find website/build -type f -name "*.html" -exec sed -i -E 's#<link[^>]*fonts\.(googleapis|gstatic)\.com[^>]*>##g' {} + && \
    find website/build -type f -name "*.html" -exec sed -i -E '/(googletagmanager|google-analytics)\.com/d' {} +

# Assert the built site loads nothing over the network. Two complementary checks:
# no <script>/<img>/<link> points at an absolute or protocol-relative URL, and no
# known asset CDN or analytics host appears anywhere in the served HTML/CSS/JS --
# the second is the only way to catch a URL inside an inline script or a lazily
# imported module, where the first cannot see.
#
# `konpyutaika.github.io` is excluded from the first check: the theme emits
# <link rel="canonical"> and og:url from the site's configured `url`, pointing at
# the authoritative public site. Those are metadata, not load-time fetches, and
# pointing them at the real site is the semantically correct thing for a mirror
# to do. Plain <a href> links in the docs are likewise left as upstream wrote
# them throughout.
RUN set -eu; \
    ! find website/build -type f \( -name '*.html' -o -name '*.css' \) -print0 \
      | xargs -0 grep -hoE '<(script|img|link|iframe)[^>]*(src|href)=["'"'"']?(https?:)?//[^"'"'"' >]+' \
      | grep -qv 'konpyutaika\.github\.io'; \
    ! find website/build -type f \( -name '*.html' -o -name '*.css' -o -name '*.js' -o -name '*.mjs' \) \
        ! -name '*.map' -print0 \
      | xargs -0 grep -qoE '(\\?/\\?/)(cdn\.jsdelivr\.net|unpkg\.com|cdnjs\.cloudflare\.com|code\.jquery\.com|ajax\.googleapis\.com|fonts\.googleapis\.com|fonts\.gstatic\.com|[a-z]+\.bootstrapcdn\.com|use\.fontawesome\.com|kit\.fontawesome\.com|www\.googletagmanager\.com|www\.google-analytics\.com|img\.shields\.io|platform\.twitter\.com|[a-z.]*gravatar\.com|ghbtns\.com|snack\.expo\.io|[a-z0-9-]*\.algolia(net)?\.com|[a-z0-9-]*\.algolia\.net)'

# The hardened nginx image serves as UID 65532, which can only read
# world-readable content. Nothing above guarantees that, so sweep the
# output: a+rX adds read for everyone and traverse on directories only,
# without marking regular files executable.
RUN chmod -R a+rX website/build

# --- Stage 2: Serve the Offline Site ---
# Red Hat Hardened Images nginx: distroless (no shell, no package manager) and
# runs as UID 65532, so this stage listens on 8080 (see default.conf), can RUN
# nothing, and must not set a CMD -- the image's own CMD carries the flags nginx
# needs after its /usr/sbin/nginx entrypoint. See README.md for the details and
# for how to verify the signatures on the published images.
FROM registry.access.redhat.com/hi/nginx:latest

COPY --from=builder /app/website/build /usr/share/nginx/html
COPY default.conf /etc/nginx/conf.d/default.conf

EXPOSE 8080
