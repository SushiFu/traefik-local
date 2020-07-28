# Traefik Local

Setup a local Traefik web proxy with DNS resolution on all \*.lan and \*.test domains.

Also setup a local trusted Root CA and create a TLS certificate for using https in local (shout out to [mkcert](https://github.com/FiloSottile/mkcert)).

This guide is for MacOS, but will work on Linux with minor modifications.

## 0. Prerequisites

- [Docker](https://docs.docker.com/docker-for-mac/install/)
- [Homebrew](https://brew.sh/)

## 1. Setup resolvers

```sh
# Setup MacOS to take into account our local docker resolver
sudo mkdir -p /etc/resolver
echo "nameserver 127.0.0.1" | sudo tee -a /etc/resolver/lan > /dev/null
echo "nameserver 127.0.0.1" | sudo tee -a /etc/resolver/test > /dev/null
```

## 2. Setup a local Root CA

```sh
brew install mkcert
brew install nss # only if you use Firefox

# Setup the local Root CA
mkcert -install

# Local Root CA files are located under ~/Library/Application\ Support/mkcert
# Look at https://github.com/FiloSottile/mkcert is you need instructions to install them on another device
```

## 3. Setup a Traefik container w/ https

```sh
# Clone this repository
git clone https://github.com/SushiFu/traefik-local.git
cd traefik-local/

# Create an external network docker, all future containers
# which need to be exposed by domain name should use this network
docker network create docker

# Create a local TLS certificate
# You could add any domain you need ending by .lan or .test
# *.this.test will create a wildcard certificate so any subdomain in the form like.this.test will also work.
# Unfortunately you cannot create *.test wildcard certificate your browser will not allow it.
mkcert -cert-file certs/local.crt -key-file certs/local.key "this.test" "*.this.test" "this.lan" "*.this.lan"

# Start Traefik
docker-compose pull
docker-compose up -d

# Go on https://traefik.this.test you should have the traefik web dashboard serve over https
```

## 4. Setup your dev containers

```sh
# On your docker-compose.yaml file

# Add the external network web at the end of the file
networks:
  default:
    external:
      name: docker

# Add these labels on the container
    labels:
      - traefik.enable=true
      - traefik.http.routers.my-frontend.entrypoints=http,https
      - traefik.http.routers.my-frontend.rule=Host(`my-frontend.this.test`) # You can use any domain allowed by your TLS certificate
      - traefik.http.routers.my-frontend.tls=true
      - traefik.http.routers.my-frontend.service=my-docker-compose-service-name
      - traefik.http.services.my-frontend.loadbalancer.server.port=3636 # Adapt to the exposed port in the service

# Protip: For web applications, use the same origin domain for your frontend and backend to avoid cookies sharing issues.
# By example: https://this.test (frontend) and https://api.this.test (backend)
```
