# Nginx Proxy docker-compose config

> This project was developed as part of my work with [VibroBox](https://github.com/vibrobox).

If you are looking for a ready-made solution for combining [jwilder/nginx-proxy](https://github.com/jwilder/nginx-proxy)
with `nginx`, which supports `brotli` compression module and automatically management of Let's Encrypt
certificates, so you are in the right place. In this repository, you can find the ready-made 
`docker-compose` configuration, which combines the following containers:

* [jwilder/docker-gen](https://github.com/jwilder/docker-gen)

  File generator that renders templates using Docker container meta-data.

* [fholzer/nginx-brotli](https://github.com/fholzer/docker-nginx-brotli)

  Nginx image with an `brotli` compression module.

* [jrcs/letsencrypt-nginx-proxy-companion](https://github.com/JrCs/docker-letsencrypt-nginx-proxy-companion)

  Companion container for the docker-gen that allows the creation/renewal of Let's Encrypt
  certificates automatically.

Successfully tested in both production and local dev environments.

## Installing

_It is worthwhile to understand that the configuration is performed once and works within the whole server._

At first, you should create a network:

```sh
docker network create nginx-proxy
```

Then you should clone this repository in any convenient place and go to its folder:

```sh
git clone https://github.com/erickskrauch/docker-compose-nginx-proxy.git nginx-proxy
cd nginx-proxy
```

After just run the containers:

```sh
docker-compose up -d
```

At this stage the setting is completed. The images will be downloaded and containers will be launched.
In the case that the server restarting, the containers will automatically start with the Docker Engine.

## Configuring projects

> The information below is largely identical to the individual manuals for using these images
  and can be studied in more details in the documentation for the respective repositories.

* [nginx-proxy guide](https://github.com/jwilder/nginx-proxy#usage).

docker-gen automatically picks up only those containers that have the environment variable `VIRTUAL_HOST`.
If you don't do this, docker-gen will not generate the configuration and nothing will work.
In addition, the container must expose at least one port (nginx and apache official images expose
80 and 443 ports by default).

In addition, you need to make sure that the container that should be proxied is available
in the network `nginx-proxy`, which we have created a bit earlier.

Below there is an example of the `docker-compose.yml` configuration for the project that should be proxied
under the name `example.com`:

```yml
version: '2'
services:
  web:
    from: nginx
    environment:
      - VIRTUAL_HOST=example.com
    networks:
      - nginx-proxy

# This is how we connect the internal network of the container with the global one, which we created earlier
networks:
  nginx-proxy:
    external:
      name: nginx-proxy
```

## SSL certificates

* [nginx-proxy guide (SSL support)](https://github.com/jwilder/nginx-proxy#ssl-support).

The required volumes have been already written in the compose file of this repository.
Following the instructions on the link above, certificates must be placed in the folder `certs`.

#### Let's Encrypt

* [letsencrypt-nginx-proxy-companion guide](https://github.com/JrCs/docker-letsencrypt-nginx-proxy-companion).

It is also possible to automatically create and further auto-update Let's Encrypt certificates.
To enable this function, it is necessary to specify two additional parameters in the environment
variables of the container along with `VIRTUAL_HOST`:

* `LETSENCRYPT_HOST` - specifies the name of the host to which the certificate is issued.
  In most cases it should be equal to `VIRTUAL_HOST` value.
 
* `LETSENCRYPT_EMAIL` - specifies the E-mail to which the certificate will be attached.
  There is no verification, you can write everything, but remember that this E-mail will
  receive notifications from Let's Encrypt in some important cases.

Certificates are generated within a couple of minutes. If this did not happen,
you can view the logs with the command:

```sh
# Execute in the folder where this docker-compose was installed
docker-compose logs -f --tail 30 letsencrypt-nginx-proxy-companion
```

I will also pay attention to the fact that certificates will be successfully issued only
if the server is actually accessible from the Internet by the specified host name.
You will not be able to write out a certificate for .local or any other non-existent
and inaccessible domain zone/domain.

To generate self-signed certificates for local development, it is convenient to use
[this service](http://www.selfsignedcertificate.com/). The file `domain.key` should be put
on the path `certs/domain.key`, and the file `domain.cert` as `certs/domain.crt`
(without `e` in the extension).

## Basic Authentication

* [nginx-proxy guide (Basic Authentication)](https://github.com/jwilder/nginx-proxy#basic-authentication-support).

* [An example of work with utility htpasswd](http://www.cyberciti.biz/faq/create-update-user-authentication-files/).

The required volumes has been already written in the compose file of this repository.
Following the instructions on the link above, files with logins and passwords must
be placed in the folder `htpasswd`.

As an example, to set Basic Authentication to host `example.com`, you must perform the
following actions (assuming that the console is opened in the `htpasswd` folder):

```sh
htpasswd -c example.com my-username
# Next, you will be prompted for the password for the user
```

If you need to add one more user, you should execute almost the same command,
only without the `-c` flag:

```sh
htpasswd example.com another-user
# Next, you will be prompted for the password for the user
```

**Important**: when you create file for the host, the docker-gen does not automatically
recreate nginx configuration, so after you created the file, you need to restart the nginx container:

```sh
docker-compose restart nginx
```
