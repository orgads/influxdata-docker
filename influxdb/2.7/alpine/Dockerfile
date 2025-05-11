FROM alpine:3.22 AS cli
RUN apk add --no-cache \
      binutils \
      curl
RUN curl -L -o influx https://output.circle-artifacts.com/output/job/e916c64c-c1b5-43b3-9288-ee6edd435b4a/artifacts/0/bin/linux/amd64/influx && \
    strip -o /usr/local/bin/influx influx && \
    rm -f influx

FROM alpine:3.22

RUN echo 'hosts: files dns' >> /etc/nsswitch.conf
RUN apk add --no-cache \
      bash \
      ca-certificates \
      curl \
      gnupg \
      openssl \
      run-parts \
      su-exec \
      tzdata && \
    update-ca-certificates

# Install dasel for configuration parsing
RUN curl -fL "https://github.com/orgads/dasel/releases/download/v2.8.1-ac-2/dasel_linux_amd64.gz" | gzip -d > /usr/local/bin/dasel && \
    chmod +x /usr/local/bin/dasel && \
    dasel --version

RUN addgroup -S -g 1000 influxdb && \
    adduser -S -G influxdb -u 1000 -h /home/influxdb -s /bin/sh influxdb && \
    mkdir -p /home/influxdb && \
    chown -R influxdb:influxdb /home/influxdb

# Install the infuxd server
ENV INFLUXDB_VERSION=2.7.12
RUN case "$(apk --print-arch)" in \
      x86_64)  arch=amd64 ;; \
      aarch64) arch=arm64 ;; \
      *) echo 'Unsupported architecture' && exit 1 ;; \
    esac && \
    export GNUPGHOME="$(mktemp -d)" && \
    gpg --batch --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys \
      # InfluxData Package Signing Key <support@influxdata.com>
      9D539D90D3328DC7D6C8D3B9D8FF8E1F7DF8B07E &&\
    curl -fLO "https://dl.influxdata.com/influxdb/releases/v${INFLUXDB_VERSION}/influxdb2-${INFLUXDB_VERSION}_linux_${arch}.tar.gz" \
         -fLO "https://dl.influxdata.com/influxdb/releases/v${INFLUXDB_VERSION}/influxdb2-${INFLUXDB_VERSION}_linux_${arch}.tar.gz.asc" && \
    gpg --batch --verify "influxdb2-${INFLUXDB_VERSION}_linux_${arch}.tar.gz.asc" \
                         "influxdb2-${INFLUXDB_VERSION}_linux_${arch}.tar.gz" && \
    tar xzf "influxdb2-${INFLUXDB_VERSION}_linux_${arch}.tar.gz" && \
    cp "influxdb2-${INFLUXDB_VERSION}/usr/bin/influxd" /usr/local/bin/influxd && \
    rm -rf "influxdb2-${INFLUXDB_VERSION}_linux_${arch}.tar.gz" \
           "influxdb2-${INFLUXDB_VERSION}_linux_${arch}.tar.gz.asc" \
           "influxdb2-${INFLUXDB_VERSION}" && \
    influxd version

# Install the influx CLI
# https://github.com/influxdata/influx-cli/pull/562
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
