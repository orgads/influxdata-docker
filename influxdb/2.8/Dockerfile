FROM alpine:3.23 AS cli
RUN apk add --no-cache binutils curl
# https://github.com/influxdata/influx-cli/pull/562
RUN curl -L -o influx https://output.circle-artifacts.com/output/job/a7cfc8f5-87c4-4e06-8389-a7ad0e886f0b/artifacts/0/bin/linux/amd64/influx && \
    strip -o /usr/local/bin/influx influx && \
    rm -f influx

FROM debian:trixie-slim

RUN export DEBIAN_FRONTEND=noninteractive && \
    # install dependencies \
    apt-get update -y && \
    apt-get install -y --no-install-recommends \
      ca-certificates \
      curl \
      gnupg \
      openssl && \
    # cleanup apt \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Install dasel for configuration parsing
RUN curl -fL "https://github.com/orgads/dasel/releases/download/v2.8.1-ac-2/dasel_linux_amd64.gz" | gzip -d > /usr/local/bin/dasel && \
    chmod +x /usr/local/bin/dasel && \
    dasel --version

RUN groupadd -r influxdb --gid=1000 && \
    useradd -r -g influxdb --uid=1000 --create-home --home-dir=/home/influxdb --shell=/bin/bash influxdb

# Install gosu for easy step-down from root
ENV GOSU_VER=1.19
RUN case "$(dpkg --print-architecture)" in \
      *amd64) arch=amd64 ;; \
      *arm64) arch=arm64 ;; \
      *) echo 'Unsupported architecture' && exit 1 ;; \
    esac && \
    export GNUPGHOME="$(mktemp -d)" && \
    gpg --batch --keyserver keyserver.ubuntu.com --recv-keys \
      # Tianon Gravi <tianon@tianon.xyz> (gosu)
      B42F6819007F00F88E364FD4036A9C25BF357DD4 && \
    curl -fLo /usr/local/bin/gosu     "https://github.com/tianon/gosu/releases/download/$GOSU_VER/gosu-${arch}" \
         -fLo /usr/local/bin/gosu.asc "https://github.com/tianon/gosu/releases/download/$GOSU_VER/gosu-${arch}.asc" && \
    gpg --batch --verify /usr/local/bin/gosu.asc \
                         /usr/local/bin/gosu && \
    rm -rf /usr/local/bin/gosu.asc && \
    chmod +x /usr/local/bin/gosu && \
    gosu --version && \
    gosu nobody true

# Install the influxd server
ENV INFLUXDB_VERSION=2.8.0
ENV INFLUXDB_PR=-2
ENV INFLUXDB_PV=${INFLUXDB_VERSION}${INFLUXDB_PR}
RUN case "$(dpkg --print-architecture)" in \
      *amd64) arch=amd64 ;; \
      *arm64) arch=arm64 ;; \
      *) echo 'Unsupported architecture' && exit 1 ;; \
    esac && \
    export GNUPGHOME="$(mktemp -d)" && \
    gpg --batch --keyserver keyserver.ubuntu.com --recv-keys \
      # InfluxData Package Signing Key <support@influxdata.com>
      24C975CBA61A024EE1B631787C3D57159FC2F927 && \
    curl -fLO "https://dl.influxdata.com/influxdb/releases/v${INFLUXDB_VERSION}/influxdb2-${INFLUXDB_PV}_linux_${arch}.tar.gz" \
         -fLO "https://dl.influxdata.com/influxdb/releases/v${INFLUXDB_VERSION}/influxdb2-${INFLUXDB_PV}_linux_${arch}.tar.gz.asc" && \
    gpg --batch --verify "influxdb2-${INFLUXDB_PV}_linux_${arch}.tar.gz.asc" \
                         "influxdb2-${INFLUXDB_PV}_linux_${arch}.tar.gz" && \
    tar xzf "influxdb2-${INFLUXDB_PV}_linux_${arch}.tar.gz" && \
    cp "influxdb2-${INFLUXDB_VERSION}/usr/bin/influxd" /usr/local/bin/influxd && \
    rm -rf "influxdb2-${INFLUXDB_PV}_linux_${arch}.tar.gz" \
           "influxdb2-${INFLUXDB_PV}_linux_${arch}.tar.gz.asc" \
           "influxdb2-${INFLUXDB_VERSION}" && \
    influxd version

# Install the influx CLI
ENV INFLUX_CLI_VERSION=2.7.5
COPY --from=cli /usr/local/bin/influx /usr/local/bin/influx
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

ENV INFLUX_CONFIGS_PATH=/etc/influxdb2/influx-configs
ENV INFLUXD_INIT_PORT=9999
ENV INFLUXD_INIT_PING_ATTEMPTS=600
ENV DOCKER_INFLUXDB_INIT_CLI_CONFIG_NAME=default
