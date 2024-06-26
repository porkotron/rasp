# Rasp

[![Docker Stars](https://img.shields.io/docker/stars/digitalidentity/rasp.svg)](https://hub.docker.com/r/digitalidentity/rasp/)

## What is this?

The [Shibboleth Service Provider](https://www.shibboleth.net/products/) is a SAML-based single
sign on (SSO) client widely deployed in academic organisations. It's used to provide access control to resources
and service for millions of staff and students around the world. The Shibboleth SP integrates very well with 
Apache HTTPd, providing sophisticated authentication and access control for web pages and web applications.

Rasp is a minimalist, Debian-based Docker image that contains the Apache web server and Shibboleth SP. It is maintained
by [Digital Identity Ltd](http://digitalidentity.ltd.uk/) and commercial support is available from [Mimoto Ltd.](http://mimoto.co.uk/)
Rasp is intended to be a solid foundation for other containers but can also be used directly by mounting volumes for
configuration directories.

This image is *not* a ready-to-use, stand-alone SP service - it's meant to be configured and then used in conjunction
with other services, or extended with additional software such as PHP or web applications. Rasp aims to be a good Docker image with careful use of layers, correct signal handling,
logging to STDOUT by default and a healthcheck.

## Why use this?

* Relatively compact (it's still quite large at 70Mb to download but we've tried our best and it's smaller than many others)
* Apache and Shibd processed are properly managed by [Runit](http://smarden.org/runit/)
* Logging is directed to STDOUT by default
* Apache and Shibboleth SP configuration files are easily customised
* Follows Docker best-practice
* Experimental: also includes mod-auth-openidc and mod-auth-cas (enabled using ENV)

## Any reasons not to use this?

* It is *not* ready-to-use, and there is no UI or simplified configuration: you need to understand how to configure
  both Apache HTTPd and Shibboleth SP software.
* It's got no warranty or support by default, but you probably weren't expecting any.
* TLS is up to you: either mount keys and configuration, or use a reverse proxy/load balancer
* Docker should not be used in production unless you have a reliable process for regularly updating images and replacing
  containers. 

## Configuring and running Rasp

## Releases

Images are available from Dockerhub and Github:

* https://hub.docker.com/repository/docker/digitalidentity/rasp
* https://github.com/orgs/Digital-Identity-Labs/packages/container/package/rasp


### Getting the image

* `docker pull digitalidentity/rasp:latest` to get the latest default version from DockerHub
* `docker pull ghcr.io/digital-identity-labs/rasp:latest"` to get the latest version from Github

### Configuring Apache and the Shibboleth SP

Run the unconfigured default SP in the foreground, with a http port available:

`docker run -it digitalidentity/rasp`

Copy the current configuration from the running container:

```bash
containerid=$(docker ps | grep rasp | awk '{print $1}')

mkdir etcfs

docker cp $containerid:/etc/apache2 ./etcfs/apache2
docker cp $containerid:/etc/shibboleth ./etcfs/shibboleth

docker stop $containerid
```

Adjust these files to suit your use-case - see the
[Shibboleth IdP documentation](https://wiki.shibboleth.net/confluence/display/IDP4/Home) for lots more information.

As you're probably copying these files over the top of existing files, you don't need to keep copies of files you aren't
changing.

### Running Rasp with your configuration

Then you can either build an image that contains your configuration, like this:

```dockerfile
FROM ghcr.io/digital-identity-labs/rasp:latest
# (Don't use latest in production)

LABEL description="An example SP image based on Rasp" \
      version="0.0.1" \
      maintainer="example@example.com"

ENV SP_HOST="sp.my-example.org"
ENV SP_URL="https://$SP_HOST"
ENV SP_ID="$SP_URL/shibboleth"

## Copy your configuration files over into the image
COPY etcfs/apache2 /etc/apache2
COPY etcfs/shibboleth /etc/shibboleth

```

or run the Rasp image with mounted directories or files:

`docker run -it -p 80:80 -v /home/bjensen/myshib/etcfs/apache2:/etc/apache2 -v /home/bjensen/myshib/etcfs/shibboleth:/etc/shibboleth digitalidentity/rasp`

### Using with Docker Compose

Running a relatively bare Rasp container on its own is only useful for some basic dev or testing work. It's far more
useful when used with other Docker containers.

For example, a `docker-compose.yml` file like this can provide a basic SP service, with TLS:

```docker-compose
version: '3'
services:
  frontend:
    image: traefik:latest
    command:
      - --api.insecure=false
      - --api.dashboard=false
      - --api.debug=false
      - --log.level=INFO
      - --providers.docker=true
      - --providers.docker.swarmMode=false
      - --providers.docker.exposedbydefault=false
      - --entrypoints.http.address=:80
      - --entrypoints.https.address=:443
      - --certificatesresolvers.letsencrypt.acme.httpchallenge=true
      - --certificatesresolvers.letsencrypt.acme.httpchallenge.entrypoint=http
      - --certificatesresolvers.letsencrypt.acme.email=pete@example.com
      - --certificatesresolvers.letsencrypt.acme.storage=/letsencrypt/acme.json
    ports:
      - "443:443"
      - "80:80"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - letsencrypt:/letsencrypt
    depends_on:
      - sp
  sp:
    image: rasp:latest
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.sp.entrypoints=https"
      - "traefik.http.routers.sp.rule=Host(`testsp.example.com`)"
      - "traefik.http.routers.sp.tls=true"
      - "traefik.http.routers.sp.tls.certresolver=letsencrypt"
      - "traefik.http.routers.sp.tls.domains[0].main=testsp.example.com"
      - "traefik.http.services.sp.loadbalancer.server.port=443"
      - "traefik.port=80"
      - "traefik.default.protocol=http"
      - "traefik.http.middlewares.tweak_sp.headers.customrequestheaders.X-Forwarded-Host=testsp.example.com"
      - "traefik.http.middlewares.tweak_sp.headers.customrequestheaders.X-Forwarded-Proto=https"
      - "traefik.http.routers.sp.middlewares=tweak_sp@docker"

volumes:
  letsencrypt:

networks:
  web:
    external: true
```

### When building an image based on Rasp

Possibly useful things to know:

* The Rasp repository contains some tests - you can copy these into your new project, and extend them.
* Certificate rebuilds will happen when the new image is built, to make sure you are not using any inherited
  keys. You can generate your own using `/etc/rasp/keygen.sh` or manually with `/usr/sbin/shib-keygen` but please
  try to keep them off the image - mount them from disk if possible.
* APACHE_EXTRA_MODS can be used to enable additional modules that are present but
  not loaded by default. Separate each module name with a space, for example:  
  `APACHE_EXTRA_MODS="auth_openidc authnz_ldap auth_cas"`
* /etc/scripts contains scripts that are run when the container starts, but before
  Apache is launched.

### When using Rasp's Dockerfile to build your own images

* SP_HOST: the host/domain name of the primary SP/website
* SP_URL: the full root URL of the primary SP/website
* SP_ID: the entity ID of the primary SP
* APACHE_EXTRA_MODS: is a list of extra modules to enable at runtime

## Other Information

### Related Projects from Digital Identity

* [Ishigaki](https://github.com/Digital-Identity-Labs/ishigaki) - An IdP base image 
* [MDQT](https://github.com/Digital-Identity-Labs/mdqt) - A tool for handling SAML metadata and MDQ services

### Why is this called "Rasp"?

This project started life as **R**ails **A**pache **SP** (RASP) but the Ruby on Rails parts have been removed 
(we're adding Rails in a derived image now, and keeping Rasp as simple as possible)

### Thanks

* We're just packaging huge amounts of work by [The Shibboleth Consortium](https://www.shibboleth.net/consortium/),
  Debian and the wider Shibboleth community. If your organisation depends on Shibboleth please consider supporting them with a
  donation.
* Github user [Porkotron](https://github.com/porkotron) for fixes and suggestions 

### Contributing

You can request new features by creating an [issue](https://github.com/Digital-Identity-Labs/rasp/issues), or submit
a [pull request](https://github.com/Digital-Identity-Labs/rasp/pulls) with your contribution.

The Rasp repo contains tests that you can use in your own projects. We're extra grateful for any contributions that
include tests.

If you have a support contract with Mimoto, please [contact Mimoto](http://mimoto.co.uk/contact/) for assistance, rather
than use Github.

### License

Copyright (c) 2017,2024 Digital Identity Ltd, UK

Licensed under the MIT License
