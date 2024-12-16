FROM alpine:3.20

RUN echo 'hosts: files dns' >> /etc/nsswitch.conf
RUN apk add --no-cache iputils ca-certificates net-snmp-tools procps lm_sensors tzdata su-exec libcap && \
    update-ca-certificates

ENV TELEGRAF_VERSION 1.33.0

RUN ARCH= && \
    case "$(apk --print-arch)" in \
        x86_64) ARCH='amd64';; \
        aarch64) ARCH='arm64';; \
        *) echo "Unsupported architecture: $(apk --print-arch)"; exit 1;; \
    esac && \
    set -ex && \
    mkdir ~/.gnupg; \
    echo "disable-ipv6" >> ~/.gnupg/dirmngr.conf; \
    apk add --no-cache --virtual .build-deps wget gnupg tar && \
    wget --no-check-certificate --no-verbose -O telegraf-${TELEGRAF_VERSION}_linux_${ARCH}.tar.gz https://output.circle-artifacts.com/output/job/df1b4737-c614-48c8-a5bf-bdf4179e9b05/artifacts/0/build/dist/telegraf-1.33.0~cdd7c89e_linux_amd64.tar.gz && \
    mkdir -p /usr/src /etc/telegraf && \
    tar -C /usr/src -xzf telegraf-${TELEGRAF_VERSION}_linux_${ARCH}.tar.gz && \
    mv /usr/src/telegraf*/etc/telegraf/telegraf.conf /etc/telegraf/ && \
    mkdir /etc/telegraf/telegraf.d && \
    cp -a /usr/src/telegraf*/usr/bin/telegraf /usr/bin/ && \
    gpgconf --kill all && \
    rm -rf *.tar.gz* /usr/src /root/.gnupg && \
    apk del .build-deps && \
    addgroup -S telegraf && \
    adduser -S telegraf -G telegraf && \
    chown -R telegraf:telegraf /etc/telegraf

EXPOSE 8125/udp 8092/udp 8094

COPY entrypoint.sh /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]
CMD ["telegraf"]
