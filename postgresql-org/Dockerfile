# --- Stage 1: Build the HTML documentation from DocBook SGML ---
# PostgreSQL's docs live in doc/src/sgml as DocBook XML and are built with the
# DocBook toolchain (xsltproc + docbook-xsl + the docbook-xml 4.5 DTD), driven
# by the postgres top-level `configure` + `make -C doc/src/sgml html`.
FROM debian:bookworm-slim AS builder

RUN apt-get update && apt-get install -y \
      git make gcc bison flex perl \
      docbook docbook-xml docbook-xsl libxml2-utils xsltproc \
      && rm -rf /var/lib/apt/lists/*

WORKDIR /pg
COPY postgres-src/ .

# `configure` is required even for a docs-only build: it is what probes for
# xmllint/xsltproc and records their paths in src/Makefile.global, which
# doc/src/sgml/Makefile includes. The --without-* flags skip optional
# dependencies (ICU, readline, zlib) that configure would otherwise require
# a full toolchain for, but that the documentation build never touches.
RUN ./configure --without-icu --without-readline --without-zlib

# Build the default (non-website) HTML style: `make html` without
# STYLE=website. The website style embeds <link>/<script> references to
# postgresql.org-hosted assets and needs network access to view; the plain
# style used here renders its own stylesheet.css and inlined SVG diagrams
# into doc/src/sgml/html/, so the result is fully self-contained.
RUN make -C doc/src/sgml html && \
    test -f doc/src/sgml/html/index.html || (echo "FATAL: no content" && exit 1) && \
    test -f doc/src/sgml/html/stylesheet.css || (echo "FATAL: no stylesheet" && exit 1)

# --- OFFLINE PATCHING: verify no external font/analytics/script references ---
# The default style build has no such references (only plain <a href> citation
# links to external sites, which are inert hyperlinks, not loaded resources).
# Fail the build if that ever changes upstream.
RUN EXTERNAL_REFS=$(grep -rhoE '<(link|script|img)[^>]*(src|href)="https?://[^"]*"' doc/src/sgml/html/*.html | grep -viE 'w3\.org' || true); \
    if [ -n "$EXTERNAL_REFS" ]; then \
      echo "FATAL: external resource reference found in generated HTML:"; echo "$EXTERNAL_REFS"; exit 1; \
    else \
      echo "OK: no external resource references"; \
    fi

# --- Stage 2: Serve the Offline Site ---
FROM nginx:alpine

COPY --from=builder /pg/doc/src/sgml/html /usr/share/nginx/html
COPY default.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
