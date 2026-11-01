# The tenant web app. Alpine variant of the same unprivileged nginx: drops perl, gnutls, libxml2 and the image codecs
# that carried most of the critical findings in docs/cve-triage.md.
FROM docker.io/nginxinc/nginx-unprivileged:1.29-alpine
# First build was refused by the scan gate: brotli-libs 1.2.0-r0, CVE-2026-111497, fixed in 1.2.1-r0.
# Patch the package rather than ignore the finding. Back to the unprivileged user afterwards.
USER root
RUN apk upgrade --no-cache brotli-libs
USER 101
COPY index.html /usr/share/nginx/html/index.html
