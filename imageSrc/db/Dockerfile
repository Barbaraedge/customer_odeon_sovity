# syntax=docker/dockerfile:1.6
FROM docker.io/postgres:17.10-alpine3.24

# Bake the healthcheck into the image so it is immune to the platform's variable
# substitution over the compose file: the script expands POSTGRES_USER/DB from
# the container environment at runtime.
COPY healthcheck.sh /usr/local/bin/healthcheck.sh
RUN chmod +x /usr/local/bin/healthcheck.sh

EXPOSE 5432
