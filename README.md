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
CONTAINER ID        IMAGE                    COMMAND                  CREATED             STATUS              PORTS                                        NAMES
1156df01e198        traefik:montdor-alpine   "/entrypoint.sh trae…"   9 minutes ago       Up 9 minutes        0.0.0.0:80->80/tcp, 0.0.0.0:8888->8080/tcp   traefik_traefik_1
```

You can also access the Traefik dashboard on [localhost:8888](http://localhost:8888).

## Access your container through a domain name

Let's image you are running a Nginx container through Docker Compose.
You would usually have at least something like that:

```yaml
version: '3.4'

services:
  nginx:
    image: 'nginx:alpine'
    ports:
      - '8080:80'
```
and access Nginx through [localhost:8080](http://localhost:8080).

With Traefik running on your computer, you use the following compose file instead:
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

**Important**: please note that, in the Traefik label, the service key (`nginx-through-traefik` in our example) must be
unique between your projects, or it will conflict and prevent you to access the container.
