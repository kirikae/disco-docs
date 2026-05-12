# --- Stage 1: Build the Static Site ---
FROM hugomods/hugo:exts-0.125.4 AS builder

USER root
WORKDIR /app
RUN git config --global safe.directory '*'

# Copy the Istio website source
COPY istio-website/ .

# 1. Install Istio's required global Node tools (extracted from their Makefile)
# They use these to natively compile their CSS and SVG sprite sheets
RUN npm install -g \
    sass \
    typescript \
    svg-symbol-sprite \
    esbuild

# 2. Install local Node dependencies
RUN npm install || true

# 3. Download Hugo Modules (Theme and shortcodes)
RUN hugo mod vendor || true

# 4. --- OFFLINE PATCHING ---
# Delete Google Analytics/Tag Manager trackers, strip Google Fonts, and remove SRI hashes
RUN find layouts -type f -name "*.html" -exec sed -i -E 's/<link[^>]*fonts.(googleapis|gstatic).com[^>]*>//g' {} + || true && \
    find . -type f -name "*.html" -exec sed -i -E '/(google-analytics|googletagmanager)/d' {} + || true && \
    find . -type f \( -name "*.html" -o -name "*.js" \) -exec sed -i -e 's/ integrity="sha[0-9]*-[^"]*"//g' -e 's/ crossorigin="anonymous"//g' {} + || true

# 5. Build the site for production
RUN hugo --minify --environment production --baseURL "/" -d /app/public

# --- Stage 2: Serve the Offline Site ---
FROM nginx:alpine

COPY --from=builder /app/public /usr/share/nginx/html
COPY default.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
