# --- Stage 1: Build the Static Site ---
FROM node:20-bookworm-slim AS builder

# Install core dependencies for downloading and compiling
RUN apt-get update && apt-get install -y \
    git \
    curl \
    ca-certificates \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

# Install Go (Strictly required for Hugo Modules to download dependencies)
ENV GO_VERSION=1.22.2
RUN curl -L "https://go.dev/dl/go${GO_VERSION}.linux-amd64.tar.gz" | tar -xz -C /usr/local
ENV PATH="/usr/local/go/bin:${PATH}"

# Install Hugo Extended
ENV HUGO_VERSION=0.125.4
RUN curl -L "https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_Linux-64bit.tar.gz" \
    | tar -xz -C /usr/local/bin hugo

WORKDIR /app

# Copy the Kubernetes website source code
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