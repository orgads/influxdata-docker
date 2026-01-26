FROM alpine:3.23 AS cli
RUN apk add --no-cache \
      binutils \
      curl
RUN curl -L -o influx https://output.circle-artifacts.com/output/job/3d5a9c63-2b01-41db-a4d7-261f29c6f8b7/artifacts/0/bin/linux/amd64/influx && \
    strip -o /usr/local/bin/influx influx && \
    rm -f influx


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
RUN case "$(dpkg --print-architecture)" in \
      *amd64) arch=amd64 ;; \
      *arm64) arch=arm64 ;; \
      *) echo 'Unsupported architecture' && exit 1 ;; \
    esac && \
    curl -fL "https://github.com/TomWright/dasel/releases/download/v2.8.1/dasel_linux_${arch}.gz" | gzip -d > /usr/local/bin/dasel && \
    case ${arch} in \
      amd64) echo '21fda0a4dc3c779c42737eca4b37e4f187d7ab91ba6301eed97b801af84a9ea2  /usr/local/bin/dasel' ;; \
      arm64) echo '2c75e63f9884c37578f48788819dda5a5a5c32ec6c4a663eefc19839f44d6291  /usr/local/bin/dasel' ;; \
    esac | sha256sum -c - && \
    chmod +x /usr/local/bin/dasel && \
    dasel --version

RUN groupadd -r influxdb --gid=1000 && \
    useradd -r -g influxdb --uid=1000 --create-home --home-dir=/home/influxdb --shell=/bin/bash influxdb

# Install gosu for easy step-down from root
ENV GOSU_VER=1.19-up-2
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
RUN case "$(dpkg --print-architecture)" in \
      *amd64) arch=amd64 ;; \
      *arm64) arch=arm64 ;; \
      *) echo 'Unsupported architecture' && exit 1 ;; \
    esac && \
    export GNUPGHOME="$(mktemp -d)" && \
    gpg --batch --keyserver keyserver.ubuntu.com --recv-keys \
      # InfluxData Package Signing Key <support@influxdata.com>
      24C975CBA61A024EE1B631787C3D57159FC2F927 && \
    curl -fLO "https://dl.influxdata.com/influxdb/releases/v${INFLUXDB_VERSION}/influxdb2-${INFLUXDB_VERSION}_linux_${arch}.tar.gz" \
         -fLO "https://dl.influxdata.com/influxdb/releases/v${INFLUXDB_VERSION}/influxdb2-${INFLUXDB_VERSION}_linux_${arch}.tar.gz.asc" && \
    gpg --batch --verify "influxdb2-${INFLUXDB_VERSION}_linux_${arch}.tar.gz.asc" \
                         "influxdb2-${INFLUXDB_VERSION}_linux_${arch}.tar.gz" && \
    tar xzf "influxdb2-${INFLUXDB_VERSION}_linux_${arch}.tar.gz" && \
    cp "influxdb2-${INFLUXDB_VERSION}/usr/bin/influxd" /usr/local/bin/influxd && \
    rm -rf "influxdb2-${INFLUXDB_VERSION}_linux_${arch}.tar.gz" \
           "influxdb2-${INFLUXDB_VERSION}_linux_${arch}.tar.gz.asc" \
           "influxdb2_linux_${arch}" && \
    influxd version

# Install the influx CLI
# https://github.com/influxdata/influx-cli/pull/562
ENV INFLUX_CLI_VERSION 2.7.5
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

ENV INFLUX_CONFIGS_PATH /etc/influxdb2/influx-configs
ENV INFLUXD_INIT_PORT 9999
ENV INFLUXD_INIT_PING_ATTEMPTS 600
ENV DOCKER_INFLUXDB_INIT_CLI_CONFIG_NAME default
