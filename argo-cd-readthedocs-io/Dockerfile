FROM python:3.12-slim AS builder

WORKDIR /app

RUN apt-get update && apt-get install -y git jq && rm -rf /var/lib/apt/lists/*
RUN git config --global --add safe.directory '*'

COPY argo-cd-repo/ .

ARG OLD_VERSIONS=""

RUN mkdir -p /out/chunk_main /out/chunk_old_1 /out/chunk_old_2 /out/chunk_old_3 /out/chunk_old_4 /out/chunk_old_5

RUN echo "[]" > /tmp/versions.json && \
    jq '.[0] = {"version": "latest", "title": "latest (main)", "aliases": []}' /tmp/versions.json > /tmp/v2.json && mv /tmp/v2.json /tmp/versions.json

RUN count=1 && for version in $OLD_VERSIONS; do \
      git fetch --depth 1 https://github.com/argoproj/argo-cd.git $version:$version || true; \
      git reset --hard && git clean -fdx && git checkout $version || true; \
      pip install --no-cache-dir -r docs/requirements.txt || true; \
      sed -i 's/^site_url:.*//g' mkdocs.yml || true; \
      V_DIR=$(echo $version | sed 's/release-/v/'); \
      if [ "$count" -eq 1 ]; then \
          mkdocs build -d /out/chunk_old_${count}/stable || exit 1; \
          cp -r /out/chunk_old_${count}/stable /out/chunk_old_${count}/$V_DIR; \
          jq ". += [{\"version\": \"stable\", \"title\": \"stable ($V_DIR)\", \"aliases\": [\"stable\"]}]" /tmp/versions.json > /tmp/v2.json && mv /tmp/v2.json /tmp/versions.json; \
          jq ". += [{\"version\": \"$V_DIR\", \"title\": \"$V_DIR\", \"aliases\": []}]" /tmp/versions.json > /tmp/v2.json && mv /tmp/v2.json /tmp/versions.json; \
      else \
          mkdocs build -d /out/chunk_old_${count}/$V_DIR || exit 1; \
          jq ". += [{\"version\": \"$V_DIR\", \"title\": \"$V_DIR\", \"aliases\": []}]" /tmp/versions.json > /tmp/v2.json && mv /tmp/v2.json /tmp/versions.json; \
      fi; \
      find /out/chunk_old_${count}/ -type f -name "*.html" -exec sed -i -E 's/<link[^>]*fonts.(googleapis|gstatic).com[^>]*>//g' {} + || true; \
      count=$((count + 1)); \
    done

RUN git reset --hard && git clean -fdx && git checkout master && \
    pip install --no-cache-dir -r docs/requirements.txt && \
    sed -i 's/^site_url:.*//g' mkdocs.yml || true && \
    mkdocs build -d /out/chunk_main || exit 1 && \
    test -f /out/chunk_main/index.html || (echo "FATAL: MkDocs generated no content for main!" && exit 1) && \
    find /out/chunk_main -type f -name "*.html" -exec sed -i -E 's/<link[^>]*fonts.(googleapis|gstatic).com[^>]*>//g' {} + || true && \
    cp /tmp/versions.json /out/chunk_main/versions.json

RUN chmod -R 755 /out

FROM nginx:alpine

RUN rm -rf /usr/share/nginx/html/*

COPY --from=builder /out/chunk_old_1/ /usr/share/nginx/html/
COPY --from=builder /out/chunk_old_2/ /usr/share/nginx/html/
COPY --from=builder /out/chunk_old_3/ /usr/share/nginx/html/
COPY --from=builder /out/chunk_old_4/ /usr/share/nginx/html/
COPY --from=builder /out/chunk_old_5/ /usr/share/nginx/html/
COPY --from=builder /out/chunk_main/ /usr/share/nginx/html/

COPY default.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
