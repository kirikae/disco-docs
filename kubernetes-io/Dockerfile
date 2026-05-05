# --- Stage 1: Build the Static Site ---
FROM node:20-alpine AS builder

# Install dependencies required for Hugo Extended and fetching code
RUN apk add --no-cache git curl libc6-compat libstdc++ build-base

# Install Hugo Extended
# The k8s website requires the "extended" version of Hugo for SCSS processing.
ENV HUGO_VERSION=0.125.4
RUN curl -L "https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_Linux-64bit.tar.gz" \
    | tar -xz -C /usr/local/bin hugo

WORKDIR /app

# Copy the Kubernetes website source code
# (This assumes GitHub Actions checks the repo out into a folder named "k8s-website")
COPY k8s-website/ .

# Install Node.js dependencies (PostCSS, Autoprefixer, etc.)
RUN npm ci

# Build the site for production
RUN hugo --minify --environment production

# --- Stage 2: Serve the Offline Site ---
FROM nginx:alpine

# Copy the built static files from the builder stage
COPY --from=builder /app/public /usr/share/nginx/html

# Expose the default HTTP port
EXPOSE 80

# Start Nginx
CMD ["nginx", "-g", "daemon off;"]