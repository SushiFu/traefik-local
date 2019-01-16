# Traefik Local

Setup a local Traefik web proxy with DNS resolution on all \*.test and \*.localhost domains.

Also setup a local trusted Root CA and create a TLS certificate for using https in local (shout out to [mkcert](https://github.com/FiloSottile/mkcert)).

This guide is for MacOS, but will work on Linux with minor modifications.

## 0. Prerequisites

-   [Docker](https://docs.docker.com/docker-for-mac/install/)
-   [Homebrew](https://brew.sh/)

## 1. Install and configure dnsmasq

```sh
brew up
brew install dnsmasq

# Setup dnsmasq config
cp $(brew list dnsmasq | grep /dnsmasq.conf$) /usr/local/etc/dnsmasq.conf
echo "address=/test/127.0.0.1" >> /usr/local/etc/dnsmasq.conf
echo "address=/localhost/127.0.0.1" >> /usr/local/etc/dnsmasq.conf

# Setup daemon configuration
sudo cp $(brew list dnsmasq | grep /homebrew.mxcl.dnsmasq.plist$) /Library/LaunchDaemons/
sudo launchctl load /Library/LaunchDaemons/homebrew.mxcl.dnsmasq.plist

# Validate our local DNS resolver is working
dig local.test @127.0.0.1
# Expected:
# ;; ANSWER SECTION:
# local.test.		0	IN	A	127.0.0.1

# If non working reload dnsmasq daemon
sudo launchctl stop homebrew.mxcl.dnsmasq
sudo launchctl start homebrew.mxcl.dnsmasq

# Setup MacOS to take into account our local resolver
sudo mkdir -p /etc/resolver
echo "nameserver 127.0.0.1" | sudo tee -a /etc/resolver/test > /dev/null
echo "nameserver 127.0.0.1" | sudo tee -a /etc/resolver/localhost > /dev/null

# Validate our custom resolver is working
ping -c 1 local.test
# Expected:
# PING local.test (127.0.0.1): 56 data bytes
# The address betwen parenthesis should be 127.0.0.1
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
git clone git@git.thepunk.tech:infra/traefik-local.git
cd traefik-local/

# Create an external network web, all future containers
# which need to be exposed by domain name should use this network
docker network create web

# Create a local TLS certificate
# You could add any domain you need ending by .test or .localhost
# *.local.test will create a wildcard certificate so any subdomain in the form this.local.test will also work.
# Unfortunately you cannot create *.test wildcard certificate your browser will not allow it.
mkcert -cert-file local-cert.pem -key-file local-key.pem "local.test" "*.local.test" "home.test" "*.home.test" "home.localhost" "*.home.localhost"

# Start Traefik
docker-compose pull
docker-compose up -d

# Go on https://traefik.local.test you should have the traefik web dashboard serve over https
```

## 4. Setup your dev containers

```sh
# On your docker-compose.yaml file

# Add the external network web at the end of the file
networks:
  web:
    external: true

# Bind each exposed container to the web network
    networks:
      - web

# Add these labels on the container
    labels:
      - traefik.enable=true
      - traefik.docker.network=web
      - traefik.backend=my-frontend # Name of your application
      - traefik.frontend.rule=Host:my-frontend.local.test # You can use any domain allowed by your TLS certificate
      - traefik.port=3636 # Adapt to the exposed port in the container

# Protip: For web applications, use the same origin domain for your frontend and backend to avoid cookies sharing issues.
# By example: https://home.test (frontend) and https://api.home.test (backend)
```
