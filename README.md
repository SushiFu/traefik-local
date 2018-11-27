# Treafik Local

Setup a localhost web proxy with DNS resolution on all .localhost domains.

MacOS only.

## 1. Install and configure dnsmasq

```sh
brew up
brew install dnsmasq

# Setup dnsmasq config
cp $(brew list dnsmasq | grep /dnsmasq.conf$) /usr/local/etc/dnsmasq.conf
echo "address=/localhost/127.0.0.1" >> /usr/local/etc/dnsmasq.conf

# Setup daemon configuration
sudo cp $(brew list dnsmasq | grep /homebrew.mxcl.dnsmasq.plist$) /Library/LaunchDaemons/
sudo launchctl load /Library/LaunchDaemons/homebrew.mxcl.dnsmasq.plist

# Validate our localhost DNS resolver is working
dig home.localhost @127.0.0.1
# Expected:
# ;; ANSWER SECTION:
# home.localhost.		0	IN	A	127.0.0.1

# If non working reload dnsmasq daemon
sudo launchctl stop homebrew.mxcl.dnsmasq
sudo launchctl start homebrew.mxcl.dnsmasq

# Setup MacOS to take into account our local resolver
sudo mkdir -p /etc/resolver
echo "nameserver 127.0.0.1" | sudo tee -a /etc/resolver/localhost > /dev/null

# Validate our custom resolver is working
ping -c 1 home.localhost
# Expected:
# PING home.localhost (127.0.0.1): 56 data bytes
# The address betwen parenthesis should be 127.0.0.1
```

## 2. Setup a Traefik container

```sh
# Clone this repository
git clone git@git.thepunk.tech:infra/traefik-local.git
cd traefik-local/

# Create an external network web, all future containers
# which need to be exposed by domain name should use this network
docker network create web

# Start Traefik
docker-compose pull
docker-compose up -d

# Go on http://traefik.localhost you should have the traefik web dashboard
```

## 3. Setup your dev containers

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
      - traefik.frontend.rule=Host:traefik.localhost # You can use any domain ending by .localhost
      - traefik.port=36363 # Adapt to the exposed port in the container

# Protip: For Web applications use the same origin domain for your frontend and backend to avoid cookies issues
# By example: http://home.localhost (frontend) and http://api.home.localhost (backend)
```
