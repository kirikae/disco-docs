# --- Stage 1: Build the Guides (Asciidoctor via Maven) ---
# "keycloak.org documentation" is actually two different builds:
#   - the keycloak.org *website* (blog, downloads, community pages) is a
#     bespoke FreeMarker/Node site built by keycloak/keycloak-web, which pulls
#     the already-released guide artifacts as versioned Maven Central jars;
#   - the actual *documentation content* (Server Admin, Server Developer,
#     Operator, Securing Applications, Getting Started, High Availability,
#     Observability, Upgrading, UI Customization, Admin REST API) lives as
#     FreeMarker-templated AsciiDoc under keycloak/keycloak's `docs/guides`
#     module and is compiled with AsciidoctorJ via a Maven reactor build
#     (`./mvnw clean install -am -pl docs/guides`) using a custom
#     `keycloak-guides-maven-plugin`.
# We build the latter directly from source: it is the realistic, self-
# contained "documentation" a Keycloak user actually wants offline, and
# skips the keycloak-web site's own external Maven Central/GitHub-release
# artifact downloads. Because `keycloak-guides-maven-plugin` depends on
# `keycloak-quarkus-server-deployment`, `-am` (also-make) pulls in a large
# slice of the Keycloak reactor -- this is a genuinely heavy JVM build, not
# a shallow doc-only compile.
FROM maven:3.9.9-eclipse-temurin-21 AS builder

WORKDIR /app

RUN git config --global --add safe.directory '*'

# The reactor pulls in keycloak-js-parent (Admin/Account console + the
# TypeScript admin-client SDK), whose codegen step shells out to Kiota, a
# .NET tool. Kiota crashes on this base image with "Couldn't find a valid
# ICU package" unless globalization is set to invariant mode -- we don't
# need locale-aware codegen for a docs build, so this is the standard,
# lightweight fix (avoids installing libicu system packages).
ENV DOTNET_SYSTEM_GLOBALIZATION_INVARIANT=true

# CI checks out keycloak/keycloak (pinned to the latest stable release tag)
# into keycloak-repo/; see build-offline-docs.yml.
COPY keycloak-repo/ .

# Build only what docs/guides needs (and its reactor dependencies), skipping
# tests/quality gates that don't matter for a docs-only offline render.
RUN ./mvnw -B -ntp \
      -am -pl docs/guides \
      -DskipTests -Dmaven.test.skip=true \
      -Dcheckstyle.skip=true -Dspotless.check.skip=true -Denforcer.skip=true \
      clean install

# --- OFFLINE PATCHING ---
# Asciidoctor's `icons: font` setting (set in docs/guides/pom.xml) emits a
# remote Font Awesome stylesheet <link> (cdnjs.cloudflare.com) on every page.
# There is no docinfo.html in docs/guides, so no analytics get injected in
# the first place. Strip the Font Awesome CDN link and any stray Google
# Fonts/analytics references that might slip in via included partials.
RUN find docs/guides/target/generated-docs -type f -name "*.html" \
      -exec sed -i -E 's#<link[^>]*cdnjs\.cloudflare\.com[^>]*font-awesome[^>]*>##g' {} + \
      -exec sed -i -E 's#<link[^>]*fonts\.(googleapis|gstatic)\.com[^>]*>##g' {} + \
      -exec sed -i -E 's#<script[^>]*(google-analytics|googletagmanager)[^>]*>.*</script>##g' {} + \
      || true

# --- Stage 2: Serve the Offline Site ---
FROM nginx:alpine

# Hand-authored landing page linking to each generated guide (there is no
# built-in top-level index across guide categories).
COPY index.html /usr/share/nginx/html/index.html

COPY --from=builder /app/docs/guides/target/generated-docs /usr/share/nginx/html/guides
COPY default.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
