FROM docker.io/library/alpine:3.22 AS git

RUN apk add --no-cache git \
    && git clone https://github.com/cert-manager/website.git 

FROM docker.io/library/node:20-slim AS build

COPY --from=git /website /website

WORKDIR /website

RUN npm install -g netlify-cli \
    && npm clean-install

RUN ./scripts/build-release

FROM docker.io/library/nginx:stable-alpine

COPY --from=build /website/out /usr/share/nginx/html