# The tenant web app. Alpine variant of the same unprivileged nginx: drops perl, gnutls, libxml2 and the image codecs
# that carried most of the critical findings in docs/cve-triage.md.
FROM docker.io/nginxinc/nginx-unprivileged:1.29-alpine
COPY index.html /usr/share/nginx/html/index.html
