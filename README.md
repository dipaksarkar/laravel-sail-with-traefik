# Traefik Local Domain Setup

## Create a docker network
We will call it "web". We are creating this network so that different docker-compose stacks can connect to each other.

```
docker network create web
```
## Create traefik container
We could do this by running a simple docker command, but in this case ware a using a small docker-compose file to configure our container.

```
# Go to traefik directory
cd traefik

# If it's the firt install of mkcert, run
mkcert -install

# Generate certificate for domain "docker.localhost", "domain.local" and their sub-domains
mkcert -cert-file certs/local-cert.pem -key-file certs/local-key.pem "docker.localhost" "*.docker.localhost" "domain.local" "*.domain.local"

# Start traefik container 
docker-compose up -d
```
After the container started successfully we can access the traefik dashboard via http://traefik.docker.localhost

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
            - "traefik.http.routers.whoami.rule=Host(`whoami.docker.localhost`)"
            # Here we define in wich network treafik can find this service
            - "traefik.docker.network=web"
            # This is the port that traefik should proxy
            - "traefik.http.services.whoami.loadbalancer.server.port=80"
            # Activation of TLS
            - "traefik.http.routers.whoami.tls=true"
        restart: always
```

Now we can access the website via http://whoami.docker.localhost

## Laravel Sail Setup
Now we are starting a laravel web project where we will add three domain for main, app and api. For example laravel-app.localhost, api.laravel-app.localhost, app.laravel-app.localhost

```
# For more information: https://laravel.com/docs/sail
version: '3'

services:
    # APP_SERVICE
    laravel-app:
        build:
            context: ./vendor/laravel/sail/runtimes/8.0
            dockerfile: Dockerfile
            args:
                WWWGROUP: '${WWWGROUP}'
        image: sail-8.0/app
        extra_hosts:
            - 'host.docker.internal:host-gateway'
        labels:
            - 'traefik.enable=true'
            # Here we have to define the URL
            # More details https://doc.traefik.io/traefik/v2.0/routing/routers/#rule
            - 'traefik.http.routers.laravel-app.rule=HostRegexp(`laravel-app.localhost`, `{subdomain:[a-z]+}.laravel-app.localhost`)'
            - 'traefik.docker.network=web'
            - 'traefik.http.services.laravel-app.loadbalancer.server.port=80'
            # Activation of TLS
            - "traefik.http.routers.whoami.tls=true"
        environment:
            WWWUSER: '${WWWUSER}'
            LARAVEL_SAIL: 1
            XDEBUG_MODE: '${SAIL_XDEBUG_MODE:-off}'
            XDEBUG_CONFIG: '${SAIL_XDEBUG_CONFIG:-client_host=host.docker.internal}'
        volumes:
            - '.:/var/www/html'
        networks:
            - sail
            - web
        depends_on:
            - mysql
    mysql:
        image: 'mysql:8.0'
        ports:
            - '${FORWARD_DB_PORT:-3306}:3306'
        environment:
            MYSQL_ROOT_PASSWORD: '${DB_PASSWORD}'
            MYSQL_DATABASE: '${DB_DATABASE}'
            MYSQL_USER: '${DB_USERNAME}'
            MYSQL_PASSWORD: '${DB_PASSWORD}'
            MYSQL_ALLOW_EMPTY_PASSWORD: 'yes'
        volumes:
            - 'sailmysql:/var/lib/mysql'
        networks:
            - sail
        healthcheck:
            test: ["CMD", "mysqladmin", "ping", "-p${DB_PASSWORD}"]
            retries: 3
            timeout: 5s
networks:
    sail:
        driver: bridge
    web:
        external: true
volumes:
    sailmysql:
        driver: local

```

## Setup .env
```
APP_URL=http://laravel-app.localhost
WEB_DOMAIN=laravel-app.localhost
APP_DOMAIN=app.laravel-app.localhost
API_DOMAIN=api.laravel-app.localhost

# Docker container service name
APP_SERVICE=laravel-app
```
