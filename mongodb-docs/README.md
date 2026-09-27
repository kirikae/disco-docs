# MongoDB documentation — not mirrored here

**Decision (2026-09-27): this repo does not build an offline mirror of the MongoDB
docs.** Use MongoDB's own offline archives instead — [GUIDE.md](GUIDE.md) shows how
to serve them from the same hardened nginx image the other sites use.

## Why

1. **Unbuildable from source outside MongoDB.** `platform/docs-site` requires
   `@mdb/flora` and `@mdb/consistent-nav`; `platform/.npmrc` pins the `@mdb` scope
   to MongoDB's private AWS CodeArtifact registry, and both packages 404 on public
   npm. The older public frontend `mongodb/snooty` has the same `.npmrc` and the
   same two dependencies; `mongodb/docs-tools` is archived. Every other site here
   builds from upstream source; this one cannot.

2. **Licensed CC BY-NC-SA 3.0 US.** Per MongoDB's own style guide
   (`content/meta/source/style-guide/style/copyrights.txt`): "MongoDB licenses all
   documentation under a Creative Commons Attribution-NonCommercial-ShareAlike 3.0
   United States License." This repo publishes public images that organisations
   pull to serve internally; whether that is permitted under **NonCommercial** is
   contested, and not a call to make on users' behalf. Everything else mirrored
   here is permissive or CC-BY, so this is a difference in kind.

3. **Already solved upstream.** MongoDB publishes prebuilt offline archives
   (`docs-manual.tar.gz` is the current Manual, 212 MB — verified). They are
   effectively airgap-ready already: relative internal links, bundled fonts,
   inline-only scripts, and a single external favicon reference. A third-party
   mirror would add almost nothing.

(1) and (2) are each sufficient on their own.

## Notes

- **No `Dockerfile` in this directory, deliberately.** `validate-pr.yml` treats any
  top-level directory containing one as a site it must build; the container recipe
  lives in [GUIDE.md](GUIDE.md) instead.
- **What would change this:** MongoDB publishing those two packages publicly, or
  relicensing without the NonCommercial term. The first alone does not resolve (2).
