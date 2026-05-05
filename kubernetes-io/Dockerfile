# --- Stage 1: Build the Static Site ---
# The 'exts' tag includes Hugo Extended, Node.js, and Go natively.
FROM hugomods/hugo:exts-0.125.4 AS builder

# Switch to root to avoid permission issues when copying 
# files checked out by the GitHub Actions runner
USER root

WORKDIR /app

# Copy the Kubernetes website source code
COPY k8s-website/ .

# Install Node.js dependencies
RUN npm ci

# Build the site for production
RUN hugo --minify --environment production --renderSegments en

# --- Stage 2: Serve the Offline Site ---
FROM nginx:alpine

# Copy the built static files from the builder stage
COPY --from=builder /app/public /usr/share/nginx/html

# Expose the default HTTP port
EXPOSE 80

# Start Nginx
CMD ["nginx", "-g", "daemon off;"]