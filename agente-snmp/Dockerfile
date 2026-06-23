FROM ubuntu:22.04

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        ca-certificates \
        iproute2 \
        procps \
        snmp \
        snmpd \
        tcpdump \
    && rm -rf /var/lib/apt/lists/*

COPY snmpd.conf /etc/snmp/snmpd.conf
COPY snmp.conf /etc/snmp/snmp.conf

EXPOSE 161/udp

CMD ["snmpd", "-f", "-Lo", "-C", "-c", "/etc/snmp/snmpd.conf"]
