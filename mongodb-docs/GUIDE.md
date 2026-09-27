# Serving MongoDB's offline docs archives from a hardened nginx image

A recipe for running MongoDB's own offline documentation bundles in a container,
for anyone who needs the docs in a disconnected environment. It uses the same
runtime image and conventions as the sites this repo does build, so the result
drops into the same environment.

Read [README.md](README.md) first for why this repo does not build and publish
such an image itself. The short version: the docs cannot be built from source
outside MongoDB, and the content is licensed **CC BY-NC-SA 3.0 US**, whose
NonCommercial term this project will not make a distribution decision about on
your behalf. Doing it yourself, for your own situation, is a decision you can
make with the facts in front of you.

> **Status: untested.** Nothing in this directory is built by CI, and the
> Containerfile below has deliberately not been run. The *inputs* to it were all
> verified by hand on 2026-09-27 — the archive downloads, its internal structure,
> the single external asset reference, and the runtime image's behaviour — but the
> file itself has never been through a build. Expect to iterate slightly on first
> use, particularly the final assertion.

## Before you start: licence obligations

The documentation is licensed
[CC BY-NC-SA 3.0 US](https://creativecommons.org/licenses/by-nc-sa/3.0/us/).
If you serve it inside your organisation, you are redistributing it, which means:

- **Attribution** — keep MongoDB's authorship visible. The archives already carry
  their own branding and footer; do not strip them.
- **ShareAlike** — any modified version you distribute carries the same licence.
- **NonCommercial** — satisfy yourself that your use is permitted. Internal
  business use of NC-licensed material is contested; if that matters to your
  organisation, ask someone qualified before deploying, or ask MongoDB directly.

The Containerfile below writes the notice into the image at
`/usr/share/nginx/html/LICENSE-DOCS.txt` so it travels with the content.

## Step 1 — find the archive URL

Archives live at:

```
https://www.mongodb.com/docs/offline/<repoName>-<gitBranchName>.tar.gz
```

Take `repoName` and `gitBranchName` from the public Snooty Data API rather than
guessing — branch naming is inconsistent between projects (`manual`, `main`,
`current`, `v1.33`):

```bash
curl -s https://snooty-data-api.mongodb.com/prod/projects \
  | jq -r '.data[] | "\(.project)\t\(.repoName)\t\([.branches[].gitBranchName] | join(","))"' \
  | column -t
```

Do **not** trust the API's own `offlineUrl` field — it is stale in both
directions. The Manual reports `offlineUrl: null` on every branch yet
`docs-manual.tar.gz` exists and works; the Kubernetes Operator advertises
archives only up to v1.26 while v1.33 is current and also downloadable.

Confirmed working on 2026-09-27:

| `DOCS_ARCHIVE_URL` | Contents | Size |
|---|---|---|
| `.../offline/docs-manual.tar.gz` | MongoDB Manual, current (8.3) | 212 MB |
| `.../offline/docs-v8.0.tar.gz` | Manual 8.0 | — |
| `.../offline/cloud-docs-main.tar.gz` | Atlas | 147 MB |
| `.../offline/docs-k8s-operator-v1.33.tar.gz` | Enterprise Kubernetes Operator | — |
| `.../offline/drivers-main.tar.gz` | Drivers | — |

No archive was available for MongoDB Controllers for Kubernetes (`docs-mck`) or
the Atlas Kubernetes Operator (`docs-atlas-operator`).

## Step 2 — verify you got a file, not a web page

This is the one thing that will bite you. `www.mongodb.com` answers **HTTP 200
with a single-page-app HTML document** for archive paths that do not exist, so a
misspelled or retired filename looks like a success and you end up serving a
MongoDB marketing page. Check the gzip magic bytes:

```bash
curl -fsSL -o docs.tar.gz https://www.mongodb.com/docs/offline/docs-manual.tar.gz
od -An -tx1 -N2 docs.tar.gz | tr -d ' \n'   # must print: 1f8b
```

The Containerfile does this check for you and fails the build if it does not hold.

## Step 3 — the Containerfile

There is no `Dockerfile` on disk in this directory on purpose: `validate-pr.yml`
treats any top-level directory containing one as a site it must build, and this
is documentation, not a site. Copy the two blocks below into a scratch directory
as `Containerfile` and `default.conf`.

The work is split across two stages because the runtime image is **distroless** —
it has no shell, so unpacking and patching cannot happen there.

```dockerfile
# --- Stage 1: unpack the archive and make it airgap-clean ---
FROM docker.io/library/debian:bookworm-slim AS builder

# Pick the project/branch you want; see Step 1.
ARG DOCS_ARCHIVE_URL=https://www.mongodb.com/docs/offline/docs-manual.tar.gz

RUN apt-get update && apt-get install -y curl ca-certificates && rm -rf /var/lib/apt/lists/*

# Fetch, then prove it really is a gzip archive. A 200 from www.mongodb.com means
# nothing (see Step 2), so the status code is not the check -- the magic bytes are.
RUN set -eu; \
    curl -fsSL -o /tmp/docs.tar.gz "${DOCS_ARCHIVE_URL}"; \
    magic=$(od -An -tx1 -N2 /tmp/docs.tar.gz | tr -d ' \n'); \
    [ "$magic" = "1f8b" ] || { \
      echo "FATAL: ${DOCS_ARCHIVE_URL} did not return a gzip archive."; \
      echo "       First bytes were '${magic}' -- most likely an HTML page, i.e."; \
      echo "       that project/branch has no offline archive. See GUIDE.md Step 1."; \
      exit 1; }

# Archive entries are './'-prefixed, so this lands the site directly in /out with
# no --strip-components. index.html is at the top level.
RUN mkdir -p /out && \
    tar -xzf /tmp/docs.tar.gz -C /out && \
    test -f /out/index.html || (echo "FATAL: archive contained no index.html" && exit 1)

# The archive is very nearly self-contained already -- prebuilt HTML, relative
# internal links, locally bundled fonts, inline-only scripts. The one remotely
# hosted asset is a favicon <link> on every page, so mirror it and repoint.
RUN set -eu; \
    mkdir -p /out/vendor; \
    curl -fsSL -o /out/vendor/favicon.ico \
      https://www.mongodb.com/docs/assets/favicon.ico \
      || echo "WARNING: could not mirror the favicon; pages will show none"; \
    find /out -type f -name '*.html' -exec sed -i \
      's#https://www\.mongodb\.com/docs/assets/favicon\.ico#/vendor/favicon.ico#g' {} +

# Carry the content licence with the content (see "Licence obligations" above).
RUN printf '%s\n' \
    'MongoDB documentation, (c) MongoDB, Inc.' \
    '' \
    'Licensed under a Creative Commons Attribution-NonCommercial-ShareAlike 3.0' \
    'United States License:' \
    '    https://creativecommons.org/licenses/by-nc-sa/3.0/us/' \
    '' \
    'Repackaged unmodified, apart from repointing one favicon reference at a local' \
    'copy, from the offline archive MongoDB publishes at:' \
    "    ${DOCS_ARCHIVE_URL}" \
    > /out/LICENSE-DOCS.txt

# Assert nothing loads over the network. Only tags that actually fetch are
# checked: <link rel="canonical">, rel="alternate" and rel="sitemap" legitimately
# reference the public site as metadata, and plain <a href> links to mongodb.com
# are left exactly as MongoDB wrote them.
RUN set -eu; \
    ! find /out -type f -name '*.html' -print0 \
      | xargs -0 grep -qoE '<(script|img|iframe)[^>]*src="https?://'; \
    ! find /out -type f -name '*.html' -print0 \
      | xargs -0 grep -qoE '<link[^>]*rel="(shortcut icon|icon|stylesheet|preload|preconnect|prefetch|manifest)"[^>]*href="https?://'

# The runtime image serves as UID 65532, which can only read world-readable
# files; a+rX adds read for everyone and traverse on directories only.
RUN chmod -R a+rX /out

# --- Stage 2: serve it ---
# Red Hat Hardened Images nginx: distroless (no shell, no package manager) and
# runs as UID 65532, so this stage listens on 8080 (see default.conf), can RUN
# nothing, and must not set a CMD -- the image's own CMD carries the flags nginx
# needs after its /usr/sbin/nginx entrypoint.
FROM registry.access.redhat.com/hi/nginx:latest

COPY --from=builder /out /usr/share/nginx/html
COPY default.conf /etc/nginx/conf.d/default.conf

EXPOSE 8080
```

And `default.conf`:

```nginx
server {
    listen 8080;
    absolute_redirect off;
    root /usr/share/nginx/html;
    index index.html index.htm;

    # Pages live at <slug>/index.html and the archive's own links point at them
    # explicitly ("./faq/index.html"), so $uri resolves most requests directly;
    # $uri/index.html covers a visitor who trims the filename off.
    #
    # Note there is no bare "$uri/" here. "$uri/index.html" already covers every
    # directory that has a landing page, while "$uri/" only adds the case of one
    # that does not -- where nginx answers 403 rather than 404, which reads
    # confusingly like the permission failure an unprivileged runtime can
    # genuinely produce. The archives ship no 404 page, so misses return nginx's.
    location / {
        try_files $uri $uri/index.html $uri.html =404;
    }
}
```

## Step 4 — build and run

```bash
podman build -t mongodb-offline-docs:manual \
  --build-arg DOCS_ARCHIVE_URL=https://www.mongodb.com/docs/offline/docs-manual.tar.gz .

podman run -d --name mongodb-docs -p 8080:8080 mongodb-offline-docs:manual
```

Then open <http://127.0.0.1:8080/>. Note the port mapping is `8080:8080`, not
`8080:80` — the hardened image runs unprivileged and cannot bind port 80.

Quick check that it is serving real docs rather than a stray marketing page:

```bash
curl -s http://127.0.0.1:8080/ | grep -o '<title>[^<]*' | head -1
```

The runtime image is distroless, so `podman exec ... sh` will not work. To poke
around inside, target the builder stage instead:

```bash
podman build --target builder -t mongodb-docs-builder . \
  && podman run --rm mongodb-docs-builder bash -c 'ls /out | head'
```

## Serving several projects or versions from one image

The archives are independent site trees with relative internal links, so they can
be mounted side by side under distinct prefixes. Repeat the fetch/extract/patch
block per archive into `/out/<prefix>` instead of `/out`, and give each its own
`ARG`. Cross-links between projects will still point at `www.mongodb.com` — they
are `<a href>` links in MongoDB's content, and rewriting them reliably is not
worth attempting, since each archive's internal path layout differs from the
live site's.

## Optional — sign the image

This repo signs everything it publishes, and you can do the same here even though
CI does not build it. With a cosign key pair:

```bash
cosign sign --key cosign.key <registry>/mongodb-offline-docs@sha256:<digest>
cosign verify --key cosign.pub <registry>/mongodb-offline-docs@sha256:<digest>
```

Sign the digest rather than a tag, and see the repo [README](../README.md) for the
airgapped verification flow (`cosign save` / `cosign verify --local-image`), which
is what lets you re-check the image after it has been carried inside.

## If the download starts failing

The `offline/` endpoint is undocumented and carries no stability guarantee; the
stale `offlineUrl` values in the Data API suggest it is not closely maintained. If
an archive disappears, re-run the Step 1 query to see what branches exist now, and
remember that a 200 response proves nothing. There is no fallback: as
[README.md](README.md) explains, building the docs from source needs private
packages from MongoDB's own registry.
