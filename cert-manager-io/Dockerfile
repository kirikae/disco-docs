# --- Stage 1: Build the Static Site ---
FROM node:20-bookworm-slim AS builder

WORKDIR /app

# 1. Install prerequisites required for native Node modules
RUN apt-get update && apt-get install -y \
    git \
    curl \
    python3 \
    make \
    g++ \
    && rm -rf /var/lib/apt/lists/*

RUN git config --global --add safe.directory '*'

# Copy the cert-manager website source
COPY cert-manager-website/ .

# 2. Install Node modules
RUN npm install

# 3. Force Next.js into Offline Static Export Mode
# Next.js defaults to an active Node server. We inject configs to force purely static HTML,
# and disable the Image Optimization API which would otherwise crash a static build.
RUN for f in next.config.js next.config.mjs; do \
      if [ -f "$f" ] && ! grep -q "output: *['\"]export['\"]" "$f"; then \
        sed -i -E "s/(const nextConfig = \{|module\.exports = \{)/\1 output: 'export', images: { unoptimized: true },/" "$f" || true; \
      fi; \
    done

# 4. Strip External Dependencies (Offline Mode)
# Remove Google Fonts or external CDNs that might cause network timeouts
RUN find . -type f \( -name "*.js" -o -name "*.jsx" -o -name "*.tsx" -o -name "*.mdx" -o -name "*.css" \) \
      -exec sed -i -E 's|https://fonts.(googleapis|gstatic).com[^"'\''` ]*||g' {} + || true && \
    find . -type f -name "*.css" -exec sed -i -E 's|@import url\([^)]+\);||g' {} + || true

# 5. Build the site
# Next.js will compile all versions natively and output them to the /app/out/ directory
RUN npm run build

# --- Stage 2: Serve the Offline Site ---
FROM nginx:alpine

# Next.js places static exports in the 'out' directory (unlike Hugo's 'public' directory)
COPY --from=builder /app/out /usr/share/nginx/html
COPY default.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]