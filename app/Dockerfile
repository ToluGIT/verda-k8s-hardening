# The tenant web app. Alpine variant of the same unprivileged nginx: drops perl, gnutls, libxml2 and the image codecs
# that carried most of the critical findings in docs/cve-triage.md.
# base pinned by digest (tag 1.29-alpine as of 2026-11-01) so the build is reproducible and the scan result stays true for the pushed image
FROM docker.io/nginxinc/nginx-unprivileged:1.29-alpine@sha256:0c79d56aee561a1d81c63f00eee5fb5fe29279560cdc55e91425133104c7fbe6
# First build was refused by the scan gate: brotli-libs 1.2.0-r0, CVE-2026-111497, fixed in 1.2.1-r0.
# Patch the package instead of adding an ignore. Back to the unprivileged user afterwards.
USER root
RUN apk upgrade --no-cache brotli-libs
USER 101
COPY index.html /usr/share/nginx/html/index.html
