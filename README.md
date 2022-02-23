# Dockerized Dehydrated with Cloudflare hook

A Docker container that uses [Dehydrated](https://github.com/dehydrated-io/dehydrated) with a [Cloudflare hook](https://github.com/walcony/letsencrypt-cloudflare-hook) to request and renew certificates.

Changes made on the deploy cert stage of the [Cloudflare hook](https://github.com/walcony/letsencrypt-cloudflare-hook) to move the certs around and reload a Docker container for [Nginx proxy](https://github.com/nginx-proxy/nginx-proxy) to pick up the certs.

Docker images build for RaspberryPi (32/64 bits).

Image on [Docker hub](https://hub.docker.com/repository/docker/driesschulten/dehydrated-pi).

## Usage

It is setup to handle normal domains and wildcard certificates for the Nginx proxy by default. As per Dehydrated docs it handles certificates automatically for a domain as such:

```
*.mydomain.com > star_mydomain_com
www.mydomain.com
sub.mydomain.com > sub_mydomain_com
```

It will rename the certificates accordingly (`<cert mount>/star_mydomain_com/fullchain.pem` to `<cert mount>/mydomain.com.crt`) so that the Nginx proxy will read them.

The container will check daily if certs need to be renewed.

## Running

Sample Docker compose:

```yaml
version: "3"

services:
  nginx-proxy:
    image: jwilder/nginx-proxy:alpine
    container_name: nginx-proxy
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/tmp/docker.sock:ro
      - ./cert-renewal/ssl:/etc/nginx/certs # Use certs

  cert-renewal:
    container_name: cert-renewal
    image: driesschulten/dehydrated-pi:latest
    restart: unless-stopped
    environment:
      - 'CF_EMAIL=<CF account mail>'
      - 'CF_KEY=<CF account key>'
      - 'DOCKER_SOCKET=/tmp/docker.sock' # Optional, defaults to /tmp/docker.sock
      - 'RELOAD_CONTAINER_NAME=nginx-proxy' # Optional, defaults to nginx-proxy
    volumes:
      - ./cert-renewal/domains.txt:/dehydrated/domains.txt:ro # Point to your domains file
      - ./cert-renewal/ssl:/dehydrated/certs # Certificate storage
      - ./cert-renewal/accounts:/dehydrated/accounts # Store the account
      - ./cert-renewal/config:/dehydrated/config # Your Dehydrated config file
      - /var/run/docker.sock:/tmp/docker.sock:ro # Docker socked
```

Add a [Dehydrated config](https://github.com/dehydrated-io/dehydrated/blob/master/docs/examples/config) as needed. And setup a [domains.txt](https://github.com/dehydrated-io/dehydrated/blob/master/docs/examples/domains.txt) with the domains that need certificates.