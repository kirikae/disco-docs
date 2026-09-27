# Documentation for Disconnected environments

This repo contains the information and files to enable the building of container images that can be used to host Documentation for OpenSource software that is otherwise available on the internet.

NOTE: It is important to note, this repo is entirely un-affiliated with any of the projects mentioned within. This repository is meant to contain some (at minimum) semi-reproducable builds of documentation, for use within internet-restricted (i.e. no internet) environments. Everything should be contained within a container image, for ease of transport within these environments. With as minimal a container image as possible (static builds, nginx to present)

## Running an image

Every site is published to `ghcr.io/<owner>/<site>-offline-docs`, tagged `latest`
and with the upstream short commit SHA the docs were built from.

```bash
podman run -d -p 8080:8080 ghcr.io/<owner>/<site>-offline-docs:latest
```

**The container listens on 8080, not 80.** The images serve from the
[Red Hat Hardened Images][hi] nginx (`registry.access.redhat.com/hi/nginx`), a
distroless image that runs as an unprivileged user and so cannot bind a
privileged port. If you have existing `-p 8080:80` invocations or Kubernetes
manifests with `containerPort: 80`, they need updating to `8080`.

[hi]: https://catalog.redhat.com/en/software/containers/search?q=hi%2F

### What the runtime image implies for a site Dockerfile

Three constraints follow from that base image, and `validate-pr.yml` checks all
of them on every PR that touches a site:

- **It is distroless** — no shell, no package manager. The final stage cannot
  `RUN` anything; whatever the docroot needs must be arranged in the builder
  stage.
- **It supplies its own entrypoint** (`/usr/sbin/nginx`) with nginx's flags as
  its `CMD`. A site must not set `CMD`: that would replace the flags and leave
  the value appended to the entrypoint as a stray argument.
- **It serves as UID 65532**, which can only read world-readable files. Builder
  stages therefore end with a `chmod -R a+rX` (or `chmod -R 755`) over the
  output, because file modes survive `COPY --from` while ownership does not.

## Verifying the images

Every published image is signed twice with [cosign][cosign], over the manifest
**digest** rather than a tag — `latest` moves with each nightly build, so a
tag-scoped signature would go stale immediately.

[cosign]: https://docs.sigstore.dev/cosign/system_config/installation/

| | Proves | Needs |
|---|---|---|
| Keyless (Sigstore) | Which repo, workflow and commit built the image | Sigstore's roots, so effectively online |
| Key pair | That the holder of this project's key released it | Only `cosign.pub`; works fully offline |

### Keyless — check the build provenance

The certificate embedded in this signature records the workflow identity, so
verification asserts *who built it*, not merely that someone signed it:

```bash
cosign verify \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  --certificate-identity-regexp '^https://github.com/<owner>/<repo>/' \
  ghcr.io/<owner>/<site>-offline-docs:latest
```

Narrow `--certificate-identity-regexp` to a single workflow file if you want to
pin which build produced the image. Each build's job summary prints the exact
command with its digest filled in.

### Key pair — check a release, including inside an airgap

```bash
cosign verify --key cosign.pub ghcr.io/<owner>/<site>-offline-docs:latest
```

`cosign.pub` lives at the root of this repo; only the matching private key can
produce a signature it accepts.

To verify **after** the image has been carried into a disconnected environment,
bring the signature with it. `cosign save` writes the image and its signatures
into an OCI layout, and `cosign verify --local-image` checks that layout with no
registry and no network at all:

```bash
# Outside: capture image + signatures together
cosign save --dir ./site-image ghcr.io/<owner>/<site>-offline-docs:latest

# Inside: verify the on-disk copy, then load it into the local registry
cosign verify --key cosign.pub --local-image ./site-image --insecure-ignore-tlog
cosign load --dir ./site-image --registry <internal-registry>/<site>-offline-docs
```

`--insecure-ignore-tlog` only waives *reaching* the public transparency log;
cosign still verifies the log's signed timestamp from the material inside the
bundle, and still verifies the signature against your key. A non-zero exit
status means the check failed — do not rely on reading the output.

A plain `skopeo copy` or `podman pull` moves the image but leaves the signature
behind, since cosign stores it under a separate `sha256-<digest>.sig` tag. Use
`cosign save`/`cosign load` as above, or `cosign copy`, to keep them together.

Each build also attaches its signature bundles to the workflow run as an
artifact (30-day retention). They are not needed for verification — the
signatures in the registry are authoritative — but they are a convenient record
of which digest a given run published.

### Enabling the key-pair signature

Builds work with only the keyless signature, so this can be set up at any time.
Generate a key pair and store it:

```bash
cosign generate-key-pair                       # prompts for a passphrase
gh secret set COSIGN_PRIVATE_KEY < cosign.key  # then delete the local cosign.key
gh secret set COSIGN_PASSWORD                  # the passphrase from above
git add cosign.pub && git commit -m 'Add cosign public key'
```

The private key is passed to cosign through `env://` and never written to the
runner's disk. Keep `cosign.key` out of the repo — `.gitignore` already excludes
it.

## Adding a site

Create `<hostname-with-dashes>/` containing a `Dockerfile` (multi-stage: build
the docs, then `COPY` the output into the hardened nginx image), a
`default.conf`, and a `.gitignore` for the upstream checkout. Then add
`.github/workflows/build-<site>.yml` modelled on an existing one — the shared
`.github/actions/build-and-push-offline-docs` action handles tagging, pushing and
signing, and needs `id-token: write` in the calling workflow's `permissions` for
the keyless signature.

Copy the whole trigger block from an existing site workflow, not just the `push`
half. Each site workflow also runs on `pull_request` — filtered to its own paths —
and passes `publish: ${{ github.event_name != 'pull_request' }}` to the shared
action. On a PR that builds the image, smoke-tests that it serves on 8080 as UID
65532, and discards it: nothing is pushed, cached or signed. On `main` and on the
nightly schedule the same call publishes and signs. `validate-pr.yml` fails a PR
that adds a site workflow without both halves, so a new site cannot land
un-built or, worse, publishing from a PR.

## CI on a pull request

Two signals, deliberately split by cost:

- **`validate-pr.yml`** — seconds. Static checks on whatever the PR touches:
  Dockerfile lint and base-image resolution, `nginx -t` against the real runtime
  image, the hardened-runtime invariants (correct base, no `RUN`/`CMD` in the
  final stage, `listen 8080`), workflow wiring, and `actionlint`. These name the
  exact rule that broke, which a build failure usually does not.
- **The per-site build workflows** — minutes. A real `docker build` of each
  affected site plus a smoke test, with publishing switched off.

A change to the shared action touches no site directory, so nothing would build
it on a PR. `metallb-io` therefore also watches
`.github/actions/build-and-push-offline-docs/**` and acts as the canary — it is
the cheapest site to build, and what is being exercised is the action rather than
anything site-specific. The action installs cosign and runs `cosign version` even
when not publishing, because a broken cosign install is exactly the kind of
failure that used to reach `main`.

PR builds reuse the published `:buildcache`, so most layers are cache hits. A PR
from a fork gets a read-only token and may not be able to read that cache; the
build still runs, it is just slower.

The overriding rule for the build itself: **the served site must make no network
requests**. Fonts, scripts, icon sprites, emoji images, analytics and avatars all
get mirrored into the image and the markup repointed at the local copies. Each
site's Dockerfile ends with assertions that fail the build if an external
reference survives; copy that pattern rather than trusting a manual review, since
the references hide in places a quick grep misses (minified attributes without
quotes, ESM `import` URLs inside inline scripts, and scripts assembled at runtime
by `createElement`).

## TODO

Look into using `chunkah` tom maximise container image layer reuse: github.com/coreos/chunkah
