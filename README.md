
## Directory structure

Create a `nginx-proxy` directory next to the the web project directories. On my computer, I have a `Sites` directory in my user root where all my projects are stored with a `.local` extension.

Inside this `nginx-proxy` directory, create a `docker-compose.yml` file and a `certs` directory. This structure looks like this:
```
.  
+-- nginx-proxy  
|   +-- docker-compose.yml  
|   +-- certs  
+-- gitlab.local  
+-- my-other-app.local  
+-- my-cool-website.local
```

## Configure the nginx-proxy Docker service

In the `docker-compose.yml` file, paste this content:
```
version: "3.1"
services:  
  nginx-proxy:  
    image: jwilder/nginx-proxy:alpine  
    ports:  
      - "80:80"  
      - "443:443"  
    volumes:  
      - ./certs:/etc/nginx/certs  
      - /var/run/docker.sock:/tmp/docker.sock:ro  
    restart: unless-stopped
networks:  
  default:  
    external:  
      name: nginx-proxy
```

This files creates a single Docker service called `nginx-proxy` which uses the `jwilder/nginx-proxy` image and share its ports `80` and `443` with the host. This service belongs to a docker-network called `nginx-proxy`. Before starting this service, we need to create the network.

## Create the Docker network

In the Terminal:

docker network create nginx-proxy

## Start the service

In the Terminal:

docker-compose up -d

This uses the `docker-compose.yml` file to launch the `nginx-proxy` service which creates and starts a container.

**The nginx-proxy is now running and listens on ports 80 and 443 (for HTTPS). If the computer reboots, the nginx-proxy will start automatically with Docker for Mac. Now, we can forget about it: there is no need to change anything in the above configuration to link a new web project.**

----------

# Configure a custom url for each web project

To make a web project work with a custom url, we need to:

-   Make its url point to `localhost`.
-   Add its Docker service to the `nginx-proxy` network.

## Make an url point to localhost

To have a custom url like `gitlab.local` pointing to `localhost`, we need to modifiy the `/etc/host` file. In the terminal, edit the `hosts` file with the `nano` texteditor.

sudo nano /etc/hosts

Navigate with the arrows to the end of the file and add the line:

127.0.0.1        gitlab.local

Type `ctrl` + `x`, then `y` to save and exit nano. Now, the custom url points to localhost.

We still need to tell the nginx proxy to route a request to this url to a specific Docker container.

## Link a web project to the nginx-proxy

We have a web project with a Docker configuration (like [this one](https://medium.com/@francoisromain/getting-started-with-docker-for-local-node-js-development-192ceca18781) for example).

To link this project to the running nginx-proxy, we need to update its own `docker-compose.yml` file (**not** the one from nginx-proxy above) with a few instructions:

**1. Environment variables**
```
version: "3"

services:
  gitlab.local:
    build:
      context: ./
      dockerfile: Dockerfile
    image: flask:0.0.1

    volumes:
      - ./code:/code/

    environment:
      FLASK_APP: /code/main.py
      VIRTUAL_HOST: gitlab.local
      VIRTUAL_PORT: 5000
    command: flask run --host=0.0.0.0
    ports:
      - 5000
    expose:
      - 5000

networks:
  default:
    external:
      name: nginx-proxy
```
`VIRTUAL_PORT` is the port the web server is listening to.

`command: flask` run on default port 5000.
In the `Dockerfile` file, paste this content:
```
FROM python:3
RUN pip install flask
```

**2. Expose ports**

services:  
  gitlab.local:  
    …  
    expose:  
      - 5000

The exposed port value is the same as the `VIRTUAL_PORT` above.

Also, we should remove any existing `PORTS` instruction as we don’t want to share the ports outside the network.

**3. Network**

networks:  
  default:  
    external:  
      name: nginx-proxy

**Now lets start the service with:**

$ cd /my-app.local  
$ docker-compose up

**The app is now listening at  `http://gitlab.local**`.**

----------

# Add HTTPS

## Create a self-signed SSL certificate for a custom domain

To create the certificates, we use the [create-ssl-certificate](https://github.com/christianalfoni/create-ssl-certificate) command line tool (by Christian Alfoni). First we install it globally with `npm i -g create-ssl-certificate`.

-   go to `/nginx-proxy/certs/`.
-   issue a certificate with `create-ssl-certificate --hostname gitlab --domain local`.
-   Rename the files `ssl.crt` and `ssl.key` to `gitlab.local.crt` and `gitlab.local.key`.

## Trust the certificate

-   Add the `git.local.crt` file to the Keychain Access app.
-   In Keychain Access, click on the certificate name to open a popup.
-   In this popup, click on the small arrow in front of `Trust`.
-   In `When using this certificate`, select `Always trust` and close the popup.

Now Chrome trusts the certificate. Firefox is a bit more picky and we have to explicitly trust the certificate when prompted.

----------

**Finally our app is listening at `https://gitlab.local`!**

Próximos Passos:
[Próximo passo em um WebServer](https://medium.com/@francoisromain/host-multiple-websites-with-https-inside-docker-containers-on-a-single-server-18467484ab95)

Referência: 
[Tutorial Base](https://medium.com/@francoisromain/set-a-local-web-development-environment-with-custom-urls-and-https-3fbe91d2eaf0)

