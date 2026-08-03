FROM alpine:3.24 AS bin
RUN apk add --no-cache \
      binutils \
      curl
RUN curl -L https://dl.influxdata.com/influxdb/releases/influxdb2-client-2.8.0-linux-amd64.tar.gz \
    | tar xz ./influx && \
    strip -o /usr/local/bin/influx influx && \
    rm -f influx
# https://github.com/influxdata/influxdb/pull/27296
RUN curl -L -o influxd https://output.circle-artifacts.com/output/job/d78deeda-8b5a-448b-a86e-4c1b5ce6fcef/artifacts/0/bin/influxd_linux_amd64/influxd && \
    strip -o /usr/local/bin/influxd influxd && \
    rm -f influxd


FROM debian:trixie-slim

RUN export DEBIAN_FRONTEND=noninteractive && \
    # install dependencies \
    apt-get update -y && \
    apt-get install -y --no-install-recommends \
      ca-certificates \
      curl \
      openssl \
      gnupg && \
    # cleanup apt \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Install dasel for configuration parsing
RUN curl -fL "https://github.com/orgads/dasel/releases/download/v2.8.1-ac-4/dasel_linux_amd64.gz" | gzip -d > /usr/local/bin/dasel && \
    chmod +x /usr/local/bin/dasel && \
    dasel --version

RUN groupadd -r influxdb --gid=1000 && \
    useradd -r -g influxdb --uid=1000 --create-home --home-dir=/home/influxdb --shell=/bin/bash influxdb

# Install gosu for easy step-down from root
ENV GOSU_VER=1.19-up-3
RUN case "$(dpkg --print-architecture)" in \
      *amd64) arch=amd64 ;; \
      *arm64) arch=arm64 ;; \
      *) echo 'Unsupported architecture' && exit 1 ;; \
    esac && \
    curl -fLo /usr/local/bin/gosu     "https://github.com/orgads/gosu/releases/download/$GOSU_VER/gosu-${arch}" && \
    chmod +x /usr/local/bin/gosu && \
    gosu --version && \
    gosu nobody true

# Install the influxd server
ENV INFLUXDB_VERSION 2.7.12
COPY --from=bin /usr/local/bin/influxd /usr/local/bin/influxd
RUN influxd version

# Install the influx CLI
ENV INFLUX_CLI_VERSION=2.8.0
COPY --from=bin /usr/local/bin/influx /usr/local/bin/influx
RUN influx version
# Create standard directories expected by the entry-point.
RUN mkdir /docker-entrypoint-initdb.d && \
    mkdir -p /var/lib/influxdb2 && \
    chown -R influxdb:influxdb /var/lib/influxdb2 && \
    mkdir -p /etc/influxdb2 && \
    chown -R influxdb:influxdb /etc/influxdb2
VOLUME /var/lib/influxdb2 /etc/influxdb2

COPY default-config.yml /etc/defaults/influxdb2/config.yml
COPY entrypoint.sh /entrypoint.sh

ENTRYPOINT ["/entrypoint.sh"]
CMD ["influxd"]

EXPOSE 8086

ENV INFLUX_CONFIGS_PATH /etc/influxdb2/influx-configs
ENV INFLUXD_INIT_PORT 9999
ENV INFLUXD_INIT_PING_ATTEMPTS 600
ENV DOCKER_INFLUXDB_INIT_CLI_CONFIG_NAME default
