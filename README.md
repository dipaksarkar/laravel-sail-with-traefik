# Traefik Local Domain Setup

## Create a docker network
We will call it "web". We are creating this network so that different docker-compose stacks can connect to each other.

```
docker network create web
```

## Create traefik container
We could do this by running a simple docker command, but in this case ware a using a small docker-compose file to configure our container.

```
version: '3'

networks:
  web:
    external: true

services:
  traefik:
    image: traefik:v2.3
    command:
      - "--log.level=DEBUG"
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
    restart: always
    ports:
      - "80:80"
      - "8080:8080" # The Web UI (enabled by --api)
    networks:
      - web
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
```

Save this file in a directory and start the container by typing

```
docker-compose up -d
```
After the container started successfully we can access the traefik dashboard via http://localhost:8080

## Website setup
Now we are starting a small web project. For example this is only a small website.

```
version: '3'

networks:
  web:
    external: true

services:
    whoami:
        # A container that exposes an API to show its IP address
        image: containous/whoami
        networks:
            - web
        # Here we define our settings for traefik how to proxy our service.
        labels:
            # This is enableing treafik to proxy this service
            - "traefik.enable=true"
            # Here we have to define the URL
            - "traefik.http.routers.whoami.rule=Host(`whoami.localhost`)"
            # Here we are defining wich entrypoint should be used by clients to access this service
            - "traefik.http.routers.whoami.entrypoints=web"
            # Here we define in wich network treafik can find this service
            - "traefik.docker.network=web"
            # This is the port that traefik should proxy
            - "traefik.http.services.whoami.loadbalancer.server.port=80"
        restart: always
```

Now we can access the website via http://whoami.localhost
