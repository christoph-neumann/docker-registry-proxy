# Docker Registry Proxy

A simple docker registry proxy that supports *both* read-only access and
authenticated access with basic auth.

The read-only registry runs on port `443` and the authenticated read-write
registry runs on port 5443.

## Quick Start

Get an SSL certificate and put the credentials in `cert.pem` and `key.pem`. If
you really want to generate self-signed cert (not recommended), then run this:

    docker run --rm -it -v $(pwd):/etc/nginx/external neumann/registry-proxy gen-cert mydomain.com

And put a copy of the `cert.pem` file at:

    /etc/docker/certs.d/mydomain.com:443/ca.crt
    /etc/docker/certs.d/mydomain.com:5443/ca.crt

Add a user:

    docker run --rm -it -v $(pwd):/etc/nginx/external neumann/registry-proxy add-user myuser

Start the registry:

    docker run -d --name registry \
    -v $(pwd)/data:/var/lib/registry \
    -e "SETTINGS_FLAVOR=local" \
    registry:2

Start the proxy:

    docker run -d  --name registry-proxy \
    -p 443:443 -p 5443:5443 \
    -v $(pwd):/etc/nginx/external \
    --link registry:registry \
    neumann/registry-proxy

Push an image. Notice you have to use port 5443 for read-write access:

    docker pull busybox:latest
    docker tag busybox:latest mydomain.com:5443/busybox:latest
    docker login mydomain.com:5443
    docker push mydomain.com:5443/busybox:latest

Pull an image. No authentication is needed for read-only access if you use port 443.

    docker pull mydomain.com:443/busybox


## Run with Docker Compose

A `docker-compose.yml` file:

    version: '2'
    
    services:
      registry:
        container_name: registry
        restart: always
        image: registry:2
        environment:
          SETTINGS_FLAVOR: local
        volumes:
          - ./data:/var/lib/registry
      registry-proxy:
        container_name: registry-proxy
        restart: always
        image: registry-proxy
        ports:
          - 443:443
          - 5443:5443
        links:
          - registry:registry
        volumes:
          - .:/etc/nginx/external
