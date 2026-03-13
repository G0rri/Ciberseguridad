FROM ubuntu:22.04

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y openssh-server rsyslog curl

RUN mkdir -p /var/run/sshd

RUN useradd -m -s /bin/bash victima && echo 'victima:123456' | chpasswd

RUN curl -L -O https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-8.12.2-amd64.deb && \
    dpkg -i filebeat-8.12.2-amd64.deb

COPY filebeat.yml /etc/filebeat/filebeat.yml
RUN chmod go-w /etc/filebeat/filebeat.yml

# Forzar la creación del archivo de logs y darle permisos de lectura
RUN touch /var/log/auth.log && chmod 644 /var/log/auth.log

# Arrancar el demonio rsyslogd de forma directa, seguido de Filebeat y SSH
CMD /usr/share/filebeat/bin/filebeat -c /etc/filebeat/filebeat.yml & /usr/sbin/sshd -D -E /var/log/auth.log
