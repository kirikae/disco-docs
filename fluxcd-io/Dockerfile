# --- Stage 1: Build the Static Site ---
FROM hugomods/hugo:exts-0.125.4 AS builder

USER root
WORKDIR /app
RUN git config --global safe.directory '*'

# Copy the FluxCD website source
COPY fluxcd-website/ .

# 1. Install Node.js dependencies (for PostCSS / Tailwind)
RUN npm install

# 2. Download Hugo Modules (Docsy theme, etc.)
RUN hugo mod vendor || true

# 3. Strip External Dependencies (Offline Mode)
# Remove Google Fonts and external SRI hashes to prevent network timeouts and browser blocking
RUN find layouts -type f -name "*.html" -exec sed -i -E 's|<link[^>]*fonts.(googleapis|gstatic).com[^>]*>||g' {} + || true && \
    find . -type f \( -name "*.html" -o -name "*.js" \) -exec sed -i -e 's/ integrity="sha[0-9]*-[^"]*"//g' -e 's/ crossorigin="anonymous"//g' {} + || true

# 4. Build the site for production using root-relative paths
RUN hugo --minify --environment production --baseURL "/" -d /app/public

# --- Stage 2: Serve the Offline Site ---
FROM nginx:alpine

COPY --from=builder /app/public /usr/share/nginx/html
COPY default.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
