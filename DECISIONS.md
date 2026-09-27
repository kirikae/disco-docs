# Open decisions

Things investigated but deliberately not acted on, recorded with enough evidence
that the decision can be made later without redoing the work.

Decisions already *taken* live next to what they affect — see
[mongodb-docs/README.md](mongodb-docs/README.md) for one of those.

---

## 1. Adopt `chunkah` for content-based image layers?

**Status: undecided, deferred.** Investigated 2026-09-27 against
[github.com/coreos/chunkah](https://github.com/coreos/chunkah) (Rust,
Apache-2.0, actively developed).

### What it is

An OCI post-processing tool: it takes a flat rootfs and re-emits it as an image
whose layers are grouped by *content* rather than by Dockerfile instruction, so
unchanged content stays in unchanged layers and clients re-pull less.

Worth knowing up front: **the `hi/nginx` base these images serve from is itself
built with chunkah** — its layers are labelled as such, and chunkah finds its RPM
database and splits it into 31 package components. The base half of every image
here is therefore already chunked. It is the docroot that is not.

### The problem it would solve

Every site copies its built docs in with a single `COPY`, which becomes one
monolithic layer. That layer is a brand-new blob on every nightly build even when
almost nothing changed, and it is re-pushed to GHCR and re-pulled by every
consumer in full.

Worst case is `k8s-offline-docs`: **7.71 GB, with the docroot as a single 7.66 GB
layer**. Nine sites ship frozen archived doc versions and share that shape
(`cloudnative-pg-io`, `docs-gitea-com`, `docs-ceph-com`, `docs-opensearch-org`,
`argo-cd-readthedocs-io`, `kubernetes-io`, `docs-percona-com`,
`konpyutaika-github-io-nifikop`, `ovn-kubernetes-io`). Three of them
(`kubernetes-io`, `argo-cd-readthedocs-io`, `docs-ceph-com`) already hand-roll a
crude version of this with `chunk_old_1..5` / `chunk_main`, which chunkah would
generalise and replace.

### Measured, on `konpyutaika-github-io-nifikop` (212 MiB docroot, 35 archived versions)

| | Layers | Largest layer |
|---|---|---|
| Today | 35 (34 shared base + 1 monolithic docroot) | **212 MiB** |
| chunkah, no hints | 34 | **202 MiB** — docroot became one `chunkah/unclaimed` layer |
| chunkah + component hints | 64 | **26 MiB** |

**Out of the box it achieves nothing.** chunkah derives components from a package
database (RPM, ALPM); a docroot has no packages, so 196 MiB of docs went into a
single catch-all layer — the same monolithic blob as today.

The win appears only once components are assigned by hand, via the
`user.component` and `user.update-interval` xattrs chunkah also reads. Tagging one
component per archived version, plus components for live docs, theme assets and
vendored libraries, dropped unclaimed content from 196 MiB to 5.8 MiB and gave:

- **158 MiB (61%) frozen** — archived versions, never re-pushed after the first time
- 66 MiB shared base
- **~37 MiB actually churning** per nightly build, against 212 MiB today

### Scorecard against the three things we wanted

- **Layer reuse — a real, large win.** The only one of the three it delivers, and
  what the original TODO actually asked for.
- **Image size — no.** chunkah is "zero diff": identical bytes, different layer
  boundaries. Measured a 2.7% *increase* (254 → 261 MiB) from per-layer overhead.
- **Build speed — no.** It is a post-processing pass, and the build cost here is
  docs generation (Astro at 5,672 pages, Hugo multi-version, Docusaurus at 3,328
  pages), not layering. The pass itself is negligible: 0.5s for 247 MiB.

### What it would cost

The clean single-pass integration is chunkah's `FROM oci:` trick, and its README
is explicit that **it will not work with Docker** — Podman/Buildah only. CI here
uses `docker/build-push-action` with buildx, and PR builds depend on the buildx
registry cache (`cache-from type=registry`). Adopting chunkah properly therefore
means moving the publish build to buildah/podman and reworking that cache. That is
the real cost, and it is larger than the chunkah change itself.

Two smaller concerns, both resolved by investigation:

- **The distroless final stage has no shell to run `setfattr` in.** Not a blocker:
  verified that `user.component` xattrs set in a *builder* stage survive
  `COPY --from` under buildah, so components can be assigned where there is a
  shell and the image keeps inheriting `hi/nginx` rather than having to flatten it.
- **Layer count.** 64 by default (`--max-layers`), against a containers-storage
  hard limit of 500. Not a concern.

Untested: whether **buildx** preserves user xattrs the same way. That matters only
for the alternative of rechunking after push instead of migrating the builder —
and that alternative also costs a full extra pull and push per build, which is
unattractive at 7.71 GB.

### If we decide to do it

1. Pilot on `kubernetes-io` alone — the biggest prize by far, one 7.66 GB blob.
   Tag one component per `v1.*` version plus one for the current version.
2. Measure real GHCR push/pull deltas over a few nightly builds before touching
   anything else.
3. Only then consider the other eight versioned sites, and retire the hand-rolled
   `chunk_old_*` bucketing in the three that have it.

### If we decide not to

Delete this section and say so, so it is not investigated a third time. The
hand-rolled `chunk_old_*` approach is a reasonable approximation and can simply be
extended to more sites.

### Related but separate: image size

If size is a goal in its own right, chunkah is the wrong tool. `k8s-offline-docs`
at 7.71 GB and `docs-gitea-com` at 910 MB (697 MB of which is generated OpenAPI
operation pages) are *content volume* decisions — how many Kubernetes versions to
ship, whether gitea's full API reference is wanted offline. A much bigger lever
than layering, and one to decide on its own terms.

---

## 2. Extend the `try_files` fix to the remaining sites?

**Status: undecided.** Two sites fixed, 30 still carrying the original pattern.

Most sites' `default.conf` ends
`try_files $uri $uri.html $uri/index.html $uri/ =404`. That trailing `$uri/` is
redundant — `$uri/index.html` already covers every directory that has a landing
page — and it is actively harmful for directories that have none: nginx tries a
directory index, autoindex is off, and it answers **403 instead of 404**.

That matters because 403 is also the signature of a docroot the UID-65532 runtime
cannot read, so it sends you hunting a permissions bug that isn't there. It has
been hit twice already: `konpyutaika-github-io-nifikop` (35 archived version roots
have no landing page) and `docs-gitea-com` (Starlight sidebar groups such as
`/administration/` and `/usage/`). Both now drop `$uri/` and add
`error_page 404 /404.html`, verified: real pages 200, index-less directories a
clean 404, no nginx errors.

The remaining 30 sites work today, so this is diagnostic quality rather than a
bug. It was left alone because it changes routing on 30 live images and deserves
its own review — now cheap to validate, since PR builds smoke-test every affected
site.

## 3. Retry the cosign install on transient failures?

**Status: undecided.** Observed once, 2026-09-27.

`sigstore/cosign-installer` fetches four URLs from GitHub. One run hit a transient
**HTTP 500** from GitHub's release-asset CDN on
`cosign-linux-amd64.sigstore.json`; the installer's own retries are immediate, so
all of them landed inside the same blip and the step failed. Re-running passed, and
all four URLs verified healthy afterwards.

With 32 builds hitting those URLs nightly, this will recur. A dependency-free
mitigation is about six lines: `continue-on-error` on the install step plus an
identical retry step gated on `steps.<id>.outcome == 'failure'`, which keeps the
action's binary verification intact and still fails hard when the problem is
persistent. The alternative is to accept the occasional re-run.
