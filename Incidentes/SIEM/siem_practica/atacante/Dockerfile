FROM alpine:latest

# Instalar el cliente SSH
RUN apk update && apk add --no-cache openssh-client

# Mantener el contenedor encendido en segundo plano
CMD ["tail", "-f", "/dev/null"]
