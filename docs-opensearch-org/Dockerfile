# syntax=docker/dockerfile:1
# --- Stage 1: Build the Static Site ---
# OpenSearch's documentation-website is a Jekyll (Ruby) site. DEVELOPER_GUIDE.md and the
# upstream CI (jekyll-build.yml) both target Ruby 3.4.5+.
FROM ruby:3.4-slim AS builder

WORKDIR /site

# build-essential/pkg-config/libcurl/libssl/libyaml/zlib: native extensions needed by the
# Gemfile's typhoeus/ethon (link checker) and other C-extension gems.
RUN apt-get update && apt-get install -y \
    build-essential \
    git \
    curl \
    pkg-config \
    libcurl4-openssl-dev \
    libssl-dev \
    libyaml-dev \
    zlib1g-dev \
    && rm -rf /var/lib/apt/lists/*

COPY opensearch-src/ .

RUN gem install bundler && \
    bundle config set --local without 'development test' && \
    bundle install

# --- OFFLINE PATCHING (pre-build) ---
#
# The upstream site is built for a live, multi-version deployment at docs.opensearch.org and
# relies on several things that don't exist (or don't make sense) in a single-version, fully
# offline build: a production `url`/`baseurl` used to construct *absolute* image/script URLs,
# Google Analytics, a CDN-hosted anchor-link script, a font-awesome CDN font path, YouTube
# iframe embeds, a live cross-origin search API, and a version-switcher populated by an external
# build pipeline. Patch all of it before running `jekyll build`.
RUN <<'BUILD_PATCHES'
set -euo pipefail

# 1. `_config.yml` sets url: "https://docs.opensearch.org" and baseurl: "/latest". Several
#    includes (home_cards.html, cards.html) build **absolute** <img src="{{site.url}}{{site.baseurl}}/images/..">
#    tags from these two values (not `relative_url`), so left as-is every icon/card image on the
#    site would silently hotlink the real docs.opensearch.org over the network. Blanking both
#    makes those same includes emit root-relative paths that resolve against this offline build,
#    served at "/". Blanking ga_tracking removes the Google Analytics gtag.js include entirely
#    (theme's head.html only emits it `if site.ga_tracking != nil`).
sed -i -E 's#^url:.*#url: ""#' _config.yml
sed -i -E 's#^baseurl:.*#baseurl: ""#' _config.yml
sed -i -E 's#^ga_tracking:.*#ga_tracking:#' _config.yml

# 2. Font Awesome 4.2.0's `_sass/font-awesome/_variables.scss` points $fa-font-path at
#    //netdna.bootstrapcdn.com by default; the actual webfont files are not vendored anywhere in
#    this repo. Vendor them locally (matching assets/fonts/<family>/ sibling layout already used
#    by the other bundled fonts) and repoint the SCSS variable at the vendored, relative path so
#    the compiled CSS's @font-face url()s resolve against this build's own /assets/fonts/.
mkdir -p assets/fonts/font-awesome
for ext in eot svg ttf woff; do
  curl -fsSL "https://cdnjs.cloudflare.com/ajax/libs/font-awesome/4.2.0/fonts/fontawesome-webfont.${ext}" \
    -o "assets/fonts/font-awesome/fontawesome-webfont.${ext}"
done
sed -i -E 's#^\$fa-font-path:.*$#\$fa-font-path:        "../fonts/font-awesome" !default;#' \
  _sass/font-awesome/_variables.scss

# 3. `_includes/head_custom.html` loads anchor-js from cdnjs.cloudflare.com whenever
#    `site.anchor_links` is set (it is, in this config). Vendor the same version locally instead
#    of stripping the feature.
mkdir -p assets/js/vendor
curl -fsSL "https://cdnjs.cloudflare.com/ajax/libs/anchor-js/4.2.0/anchor.min.js" \
  -o assets/js/vendor/anchor.min.js

# 4. Rewrite head_custom.html:
#    - anchor-js now served from the vendored copy above, not cdnjs.
#    - the MathJax-via-jsdelivr block is dropped: no page in this repo sets `has_math: true`
#      (verified against the source), so it's dead weight, and vendoring the full MathJax
#      distribution for zero call sites isn't worth the image size. Left in place it would still
#      be a live offline-CDN pitfall the moment a page enables it, so it's removed rather than
#      left dormant.
#    - the dev/prod branch that pointed at `/latest/assets/js/version-selector.js` (a path that
#      only exists in OpenSearch's separate production deploy pipeline, never in this repo) is
#      replaced with a single reference to a local stub shipped in step 6 below.
cat > _includes/head_custom.html <<'HEAD_CUSTOM'
<!-- Inline theme detection to prevent dark mode flash -->
<script>
  (function() {
    const THEME_KEY = 'opensearch-docs-theme';
    const DARK_THEME = 'dark';

    function getStoredTheme() {
      try {
        return localStorage.getItem(THEME_KEY);
      }
      catch (e) {
        return null;
      }
    }

    function getSystemTheme() {
      if (window.matchMedia && window.matchMedia('(prefers-color-scheme: dark)').matches) {
        return DARK_THEME;
      }
      return 'light';
    }

    function getPreferredTheme() {
      const stored = getStoredTheme();
      if (stored) {
        return stored;
      }
      return getSystemTheme();
    }

    const theme = getPreferredTheme();
    if (theme === DARK_THEME) {
      document.documentElement.setAttribute('data-skin', 'dark');
    }
    else {
      document.documentElement.removeAttribute('data-skin');
    }
  })();
</script>

{% if site.doc_version != "latest" or page.url == "/search.html" or page.url == "/404.html" %}
<meta name="robots" content="noindex, follow">
{% endif %}

<!-- OFFLINE PATCH: vendored locally, was cdnjs.cloudflare.com/.../anchor-js/... -->
{% if site.anchor_links != nil %}
  <script src="{{ '/assets/js/vendor/anchor.min.js' | relative_url }}"></script>
{% endif %}

<!-- OFFLINE PATCH: MathJax-via-jsdelivr removed; unused by any page in this build -->

<!-- OFFLINE PATCH: single-version build, use the local stub instead of the production
     /latest/assets/js/version-selector.js path, which does not exist in this repo -->
<script src="{{ '/assets/js/version-selector.js' | relative_url }}"></script>
HEAD_CUSTOM

# 5. `_includes/youtube-player.html` renders a bare YouTube <iframe>, which pulls in
#    youtube.com/google tracking/ad hosts as soon as the page loads. Replace it with an inert
#    link the reader can follow (and fails gracefully, exactly like any other external hyperlink,
#    if this offline build has no network access).
cat > _includes/youtube-player.html <<'YOUTUBE'
<div class="embed-container">
  <a class="video-embed-offline-link" href="https://www.youtube.com/watch?v={{ include.id }}" target="_blank" rel="noopener noreferrer">
    &#9654; Watch this video on YouTube (video embeds are disabled in this offline build): youtube.com/watch?v={{ include.id }}
  </a>
</div>
YOUTUBE

# 6. `assets/js/search.js` calls fetch() against https://search-api.opensearch.org for both the
#    header quick-search box and the /search.html results page (window.doResultsPageSearch).
#    That API is unreachable offline. Replace it with a stub that keeps the same DOM
#    hooks/classes (so the theme's CSS still applies) but shows an inline "unavailable" message
#    instead of making a cross-origin network call.
cat > assets/js/search.js <<'SEARCHJS'
// OFFLINE BUILD: the upstream implementation fetches search results from
// https://search-api.opensearch.org, which is unreachable in an offline/air-gapped
// environment. This stub disables the network call and tells the reader search isn't
// available, instead of leaving a silently-failing fetch() in place.
(() => {
    document.addEventListener('DOMContentLoaded', () => {
        const elInput = document.getElementById('search-input');
        const elResults = document.getElementById('search-results')?.querySelector?.('.top-banner-search--field-with-results--field--wrapper--search-component--search-results-wrapper') ?? null;
        const elOverlay = document.querySelector('.top-banner-search--overlay');
        if (!elInput || !elResults || !elOverlay) return;

        const showUnavailable = () => {
            elResults.replaceChildren();
            const message = document.createElement('div');
            message.classList.add('search-page--results--no-results');
            message.textContent = 'Search is unavailable in this offline documentation build.';
            elResults.appendChild(message);
            elResults.setAttribute('aria-expanded', 'true');
            document.documentElement.classList.add('search-active');
        };

        const hide = () => {
            elResults.setAttribute('aria-expanded', 'false');
            document.documentElement.classList.remove('search-active');
        };

        elInput.addEventListener('input', () => {
            if (elInput.value.trim().length >= 2) showUnavailable();
            else hide();
        });

        elInput.addEventListener('keydown', e => {
            if (e.key === 'Escape' || e.key === 'Esc') {
                hide();
                elInput.value = '';
            }
        });

        elInput.addEventListener('focus', () => {
            if (elInput.value.trim().length >= 2) showUnavailable();
        });

        document.body.addEventListener('pointerdown', e => {
            if (!e.target.closest('.top-banner-search--field-with-results')) hide();
        });
    });
})();

window.doResultsPageSearch = async () => {
    const searchResultsContainer = document.getElementById('searchPageResultsContainer');
    if (!searchResultsContainer) return;
    searchResultsContainer.innerHTML = '';
    const message = document.createElement('div');
    message.textContent = 'Search is unavailable in this offline documentation build.';
    searchResultsContainer.appendChild(message);
};
SEARCHJS

# 7. Production OpenSearch deploys generate assets/js/version-selector.js from
#    assets/js/_version-selector.js, templated with a DOC_VERSIONS global from a build pipeline
#    that lives outside this repo. Since head_custom.html (patched above) always references
#    /assets/js/version-selector.js, ship a minimal local stub that defines the same
#    <version-selector> custom element as a static, non-networked label (this is a single-version
#    build, so there is nothing to switch between) instead of leaving the reference 404ing.
cat > assets/js/version-selector.js <<'VERSIONJS'
// OFFLINE BUILD: replaces the upstream multi-version <version-selector> custom element (which is
// populated by a DOC_VERSIONS global injected by OpenSearch's production build pipeline, not
// present here, and links to sibling version paths that don't exist in this single-version
// offline build) with a static, non-interactive label.
class VersionSelector extends HTMLElement {
    connectedCallback() {
        this.attachShadow({ mode: 'open' });
        const label = document.createElement('span');
        label.textContent = `OpenSearch ${this.getAttribute('selected') || ''}`.trim();
        label.style.cssText = 'display:inline-block;padding:0.3em 1em;border-radius:4px;background:#00509c;color:#fff;';
        this.shadowRoot.appendChild(label);
    }
}
customElements.define('version-selector', VersionSelector);
VERSIONJS

# assets/js/version-selector.tpl carries its own front matter (permalink: /assets/js/version-selector.js)
# that makes Jekyll render *it* to the exact same output path as the static stub above (a
# DOC_VERSIONS-templated file wired to sibling /2.19/, /3.6/, etc. version paths that don't exist
# in this single-version build). Remove it so the stub from this build is the only thing that can
# land at that path, instead of leaving a non-deterministic "which one wins" conflict.
rm -f assets/js/version-selector.tpl
BUILD_PATCHES

# Build the single "latest" version (OpenSearch 3.7, per _data/versions.json's "current"/"latest"
# at the time this was written) at the site root.
RUN JEKYLL_ENV=production bundle exec jekyll build --future -d /out && \
    test -f /out/index.html || (echo "FATAL: no content" && exit 1)

# A small number of content pages hotlink screenshots directly from
# user-images.githubusercontent.com (pasted GitHub image-upload URLs) instead of committing them
# under /images/. Vendor exactly those images and rewrite the pages to reference the local copy,
# instead of leaving a live cross-origin image fetch in otherwise-offline pages.
RUN <<'VENDOR_HOTLINKED_IMAGES'
set -euo pipefail
mkdir -p /out/assets/images/vendor
urls=$(grep -rhoE 'https://user-images\.githubusercontent\.com/[^"'"'"')\s]+' /out --include='*.html' | sort -u)
for url in $urls; do
  name=$(basename "$url")
  curl -fsSL "$url" -o "/out/assets/images/vendor/${name}"
  escaped_url=$(printf '%s\n' "$url" | sed -E 's#[.[\*^$/]#\\&#g')
  find /out -type f -name "*.html" -exec sed -i "s#${escaped_url}#/assets/images/vendor/${name}#g" {} +
done
VENDOR_HOTLINKED_IMAGES

# Defense-in-depth: strip any Google Fonts/Analytics markup that might have slipped through.
RUN find /out -type f -name "*.html" -exec sed -i -E 's#<link[^>]*fonts\.(googleapis|gstatic)\.com[^>]*>##g' {} + && \
    find /out -type f -name "*.html" -exec sed -i -E 's#<script[^>]*google-analytics[^>]*>.*</script>##g' {} + && \
    find /out -type f -name "*.html" -exec sed -i -E 's#<script[^>]*googletagmanager\.com[^>]*></script>##g' {} +

# Homepage cards + a few in-page links use ROOT-RELATIVE `/latest/<component>/`
# paths (benchmark, clients, data-prepper, migration-assistant, and a bare
# `/latest/`), because upstream deploys the whole site under a `/latest/` path
# prefix in production. This single-version build serves that same content at
# the ROOT (`/clients/`, `/benchmark/`, ...), so those links 404. Rewrite the
# root-relative `/latest/` prefix to `/`. This deliberately does NOT touch the
# absolute `https://docs.opensearch.org/latest/...` top-nav hyperlinks (they
# start with `href="https://`, not `href="/latest/`). Guard that none survive.
RUN find /out -type f -name "*.html" -exec sed -i -E 's#(href|src)="/latest/#\1="/#g' {} + && \
    ! grep -rqE '(href|src)="/latest/' /out || (echo "FATAL: root-relative /latest/ link survived" && exit 1)

RUN chmod -R 755 /out

# --- Stage 2: Serve the Offline Site ---
FROM nginx:alpine

COPY --from=builder /out /usr/share/nginx/html
COPY default.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
