# --- Stage 1: Build the Static Site ---
FROM klakegg/hugo:0.107.0-ext-ubuntu AS builder

WORKDIR /src

RUN rm -f /etc/apt/sources.list.d/nodesource.list && \
    apt-get update && \
    apt-get install -y git build-essential python3-venv python3-pip curl && \
    rm -rf /var/lib/apt/lists/*

RUN curl -fsSL https://deb.nodesource.com/setup_22.x | bash - && \
    apt-get install -y nodejs && \
    npm install -g npm@10.9.2

RUN git config --global --add safe.directory '*'

COPY tekton-website/ .

RUN npm install

RUN python3 -m venv .venv && \
    . .venv/bin/activate && \
    pip3 install -r requirements.txt && \
    ./sync/sync.py

RUN hugo --minify --environment production --baseURL "/" -d /src/public

# --- Stage 2: Serve the Offline Site ---
FROM nginx:alpine

COPY --from=builder /src/public /usr/share/nginx/html
COPY default.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]