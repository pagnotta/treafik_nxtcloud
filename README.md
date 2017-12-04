# Nextcloud served on Traefik and Docker

## Preface
Becoming interested in Docker and Microservices I decided to give it a shot and replace my classic Nginx/Nextcloud installation by something containerized.

In my research I came across [traefik](https://github.com/containous/traefik). It acts not only as a HTTP [reverse proxy](https://en.wikipedia.org/wiki/Reverse_proxy), and as a [load balancer](https://en.wikipedia.org/wiki/Load_balancing_(computing)). It can also dynamically detect running microservices and create routes to them. One of my personal favourites is also, that it can automatically install Let's Encrypt certificates and route all traffic through https.

As a starter I studied the excellent DigitalOcean tutorial on a traefic/WordPress setup:

https://www.digitalocean.com/community/tutorials/how-to-use-traefik-as-a-reverse-proxy-for-docker-containers-on-ubuntu-16-04

(Kudos at this point for DigitalOcean, for maintaining an excellent source of information with this blog)

I departed from this tutorial, to finally come up with a traefic/Nextcloud setup, including the MySQL database and an adminer container.

## Prerequisites
* A domain
* Ubuntu Server 16.04
* Docker
    * https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-16-04
* Docker-Compose (handles the containers)
    * https://www.digitalocean.com/community/tutorials/how-to-install-docker-compose-on-ubuntu-16-04

## Installation
First of all create a project folder and go inside.

```
mkdir traefik
cd traefik
```

### traefik configuration
Before traefik is installed a passord hash needs to be generated, which will serve as the admin password for traefik.

```
sudo apt-get install apache2-utils
htpasswd -nb admin secure_password
```
Replace `secure_password` with your password you want to use for traefik. Copy the output, which you will use inside your traefic configuration.

Traefik is configured by a `traefik.toml` file
. Create it and edit it with

```
nano traefik.toml
```
The contents are:

```toml
defaultEntryPoints = ["http", "https"]
[web]
address = ":8080"
  [web.auth.basic]
  users = ["admin:passwordhash"]
[entryPoints]
  [entryPoints.http]
  address = ":80"
    [entryPoints.http.redirect]
      entryPoint = "https"
  [entryPoints.https]
  address = ":443"
    [entryPoints.https.tls]
[acme]
email = "your@email.address"
storage = "acme.json"
entryPoint = "https"
onHostRule = true
onDemand = false
```

Replace `passwordhash` with the output you copied in the previous step, and enter your email address, which will be used for the Let's encrypt certificate generation.

The ssl certificate information will be stored inside `acme.json`. Therefore we need to create it:

```
touch acme.json
chmod 600 acme.json
```

### docker-composer configuration

Docker-composer will take care of starting up the docker containers in the right order with the right configuration.

First a docker network named `proxy` is created which will serve internally for the docker containers:

`docker network create proxy`

Then docker-composer is configured using a `docker-compose.yml` file:

```
nano docker-compose.yml
```

Its contents are:

```yaml
version: "3"

networks:
  proxy:
    external: true
  internal:
    external: false

volumes:
  nextcloud:
  apps:
  data:
  config:
  mysqldir:

services:
  traefik:
    image: traefik:1.4.4-alpine
    command: --docker
    container_name: traefik
    hostname: traefik
    labels:
      - traefik.frontend.rule=Host:monitor.mydomain.com
      - traefik.port=8080
    restart: always
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /path/to/traefik/traefik.toml:/traefik.toml
      - /path/to/traefik/acme.json:/acme.json
    networks:
      - proxy
    ports:
      - 80:80
      - 443:443

  nextcloud:
    image: nextcloud
    environment:
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=root
    links:
      - mysql
    restart: always
    volumes:
      - nextcloud:/var/www/html
      - apps:/var/www/html/custom_apps
      - data:/var/www/html/data
      - config:/var/www/html/config
    labels:
      - traefik.backend=nextcloud
      - traefik.frontend.rule=Host:nextcloud.mydomain.com
      - traefik.docker.network=proxy
      - traefik.port=80
    networks:
      - internal
      - proxy
    depends_on:
      - mysql

  mysql:
    image: mariadb:latest
    environment:
      MYSQL_ROOT_PASSWORD:
    restart: always
    volumes:
      - mysqldir:/var/lib/mysql
    networks:
      - internal
    labels:
      - traefik.enable=false
 
  adminer:
    image: adminer:4.3.1-standalone
    labels:
      - traefik.backend=adminer
      - traefik.frontend.rule=Host:db-admin.mydomain.com
      - traefik.docker.network=proxy
      - traefik.port=8080
    networks:
      - internal
      - proxy
    depends_on:
      - mysql

```

Replace all three `mydomain.com` occurences with your domain, and adjust the `/path/to/` in the traefik volumes.

As you can see, traefik itself is coming as a docker container :)

Unlike in the DigitalOcean tutorial, I included the traefik container inside the docker-composer configuration file. Like that the whole setup can comfortably be started with one single command.

Next important step before starting, is to set the `MYSQL_ROOT_PASSWORD` variable with:

`export MYSQL_ROOT_PASSWORD=secure_database_password`

Of course with replacing `secure_database_password` with your secure password. This password will be used to login to MySQL using adminer.

### Starting

Time to run it.

`docker-compose up -d`

This should install and start all the containers specified in the `docker-compose.yml`. If there are problems it can be useful to run just `docker-compose up` which will give output on the console.

Now open up

https://monitor.mydomain.com

(This might take a while, since traefik is going to install the Let's Encrypt certificates after the first start.)

You should be asked to enter your traefik credentials. Don't enter your hashed password, but your plain password here ;)

You should see now the admin panel of traefik.

### Generate database

Open adminer:

https://db-admin.mydomain.com

Here enter your root user credentials and use the `mysql` server. Inside create a `nextcloud` database.

### Setup Nextcloud
Finally to setup Nextcloud open

https://nextcloud.mydomain.com

Choose MySQL as the database. In the database fields enter your root user credentials, and for the server use `mysql:3306`. If everything goes well, Nextcloud will generate your user and a MySQL user, and you should see soon the Nextcloud welcome screen.

### Stopping
If you want to stop the system a

`docker-compose stop`

will do.

## Links
Useful Links were

* Tuts
  * https://www.digitalocean.com/community/tutorials/how-to-use-traefik-as-a-reverse-proxy-for-docker-containers-on-ubuntu-16-04
  * https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-16-04
  * https://www.digitalocean.com/community/tutorials/how-to-install-docker-compose-on-ubuntu-16-04
  * https://www.digitalocean.com/community/tutorials/how-to-remove-docker-images-containers-and-volumes
* treafik docu
  * https://github.com/containous/traefik
  * https://docs.traefik.io/
* docker-compose docu
  * https://docs.docker.com/compose/
* containers used
  * traefik
    * https://hub.docker.com/r/_/traefik/
  * Nextcloud
    * https://hub.docker.com/_/nextcloud/
    * https://github.com/nextcloud/docker
  * MariaDB
    * https://hub.docker.com/_/mariadb/
  * Adminer
    * https://hub.docker.com/_/adminer/
