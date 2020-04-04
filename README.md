# Traefik as local reverse proxy

This repository provides everything needed to run Traefik through Docker Compose
so you can access your Docker containers through a domain name instead of binding Docker ports.

## Install and run Traefik on your machine

*The following instructions assume Traefik will run from `/srv/traefik` directory, on a Docker network named `proxy`,*
*but this is not mandatory. Feel free to change that at your liking.*

First copy the following files in your system by running the following commands:
```bash
$ git clone https://github.com/damien-carcel/traefik-as-local-reverse-proxy.git && cd traefik-as-local-reverse-proxy
$ sudo mkdir /srv/traefik
$ sudo cp traefik/docker-compose.yaml /srv/traefik
$ sudo cp traefik/traefik.yaml /srv/traefik
$ sudo cp traefik/docker-compose-traefik.service /etc/systemd/system/
```

Then create the Docker network Traefik will run on:
```bash
$ docker network create proxy
```

And finally start the SystemD service and enable it so it always start when you boot your computer:
```bash
$ sudo systemctl start docker-compose-traefik.service
$ sudo systemctl enable docker-compose-traefik.service
```

## Check that Traefik is running
```bash
$ docker container ls --filter=name=traefik
```

You should see something like that:
```
CONTAINER ID        IMAGE           COMMAND                  CREATED             STATUS              PORTS                                        NAMES
xxxxxxxxxxxx        traefik         "/entrypoint.sh trae…"   2 seconds ago       Up 1 second         0.0.0.0:80->80/tcp, 0.0.0.0:8888->8080/tcp   traefik_traefik_1
```

You can also access the Traefik dashboard on [localhost:8888](http://localhost:8888).

## Access your container through a domain name

Let's run a Nginx container through Docker Compose. You would usually have at least something like the following compose
file, allowing you to access Nginx through [localhost:8080](http://localhost:8080):
```yaml
version: '3.4'

services:
  nginx:
    image: 'nginx:alpine'
    ports:
      - '8080:80'
```

With Traefik running on your computer, you can use the following compose file instead:
```yaml
version: '3.4'

services:  
  nginx:
    image: 'nginx:alpine'
    labels:
      - 'traefik.http.routers.nginx-through-traefik.rule=Host(`nginx.docker.localhost`)'
    networks:
      - proxy

networks:
  proxy:
    external: true
```
and access Nginx through [nginx.docker.localhost](http://nginx.docker.localhost).

There is no need to edit your `/etc/hosts` file as we are using `localhost` as top level domain.

**Important**: please note that the router name (`nginx-through-traefik` in our example) can be whatever you want, but
it **must be unique for each of your projects**, or it will conflict and prevent you to access the container.

## Use HTTPS with self-signed certificate

### Generate a certificate

First install [mkcert](https://github.com/FiloSottile/mkcert) to generate SSL certificates.

Then install its its local CA in the system trust store, so you web browsers will trust the certificates you generate:
```bash
$ mkcert -install
```

You can now generate a wildcard certificate for `docker.localhost` domain:
```bash
$ mkcert "*.docker.localhost"
$ sudo mkdir /srv/traefik/ssl
$ sudo cp _wildcard.docker.localhost* /srv/traefik/ssl
```

This generated a certificate `_wildcard.docker.localhost.pem` and its key `_wildcard.docker.localhost-key.pem`.
It is important to generate the certificate with the same user you deployed the root CA previously.

### Setup Traefik with HTTPS

Start by replacing Traefik `docker-compose.yaml` with the one ready for HTTPS (it already expects the SSL certificates
to be in `/srv/traefik/ssl`):
```bash
$ sudo cp -r traefik/dynamic /srv/traefik
$ sudo cp traefik/docker-compose.https.yaml /srv/traefik/docker-compose.yaml
```

Then restart the SystemD service:
```bash
$ sudo systemctl restart docker-compose-traefik.service
```

Listing the Traefik container should now output something like:
```
CONTAINER ID        IMAGE           COMMAND                  CREATED             STATUS              PORTS                                                              NAMES
xxxxxxxxxxxx        traefik         "/entrypoint.sh trae…"   2 seconds ago       Up 1 second         0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp, 0.0.0.0:8888->8080/tcp   traefik_traefik_1
```

### Enable Traefik on your application

All you need now is to add a few more labels to the previous Docker Compose example:
```yaml
services:
  nginx:
    labels:
      # This label was already there
      - 'traefik.http.routers.nginx-through-traefik.rule=Host(`nginx.docker.localhost`)'
      # And below are the new labels
      - 'traefik.http.middlewares.redirect-nginx-through-traefik-http-to-https.redirectScheme.scheme=https'
      - 'traefik.http.routers.nginx-through-traefik.entrypoints=web'
      - 'traefik.http.routers.nginx-through-traefik.middlewares=redirect-nginx-through-traefik-http-to-https'
      - 'traefik.http.routers.nginx-through-traefik-with-https.entrypoints=websecure'
      - 'traefik.http.routers.nginx-through-traefik-with-https.rule=Host(`nginx.docker.localhost`)'
      - 'traefik.http.routers.nginx-through-traefik-with-https.tls=true'
```

You should now be able to access Nginx through [https://nginx.docker.localhost](https://nginx.docker.localhost).
What's more, thanks to the `redirect-nginx-through-traefik-http-to-https` middleware declared through the second label,
you are now redirected directly from HTTP to HTTPS.

**Important**: like in the first use case, the router names (in this example :`nginx-through-traefik` for HTTP,
`nginx-through-traefik-with-https` for HTTPS) and the middleware name (`redirect-nginx-through-traefik-http-to-https`)
can be whatever you want, but **they must be unique for each of your projects**.
