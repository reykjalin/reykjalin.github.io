---
layout: post
title: Deploying WordPress using Docker
---
## Initial Points  
* You can find the [WordPress](https://wordpress.com) [Docker](https://docker.com) container [here](https://hub.docker.com/_/wordpress/)
* If you’re using Docker, use the Docker images built to make your life easier! nginx-proxy and letsencrypt-nginx-proxy-companion made everything so much easier!
* I used [Docker Compose](https://docs.docker.com/compose/) to set up my Docker containers, instead of running them all individually.
* I saved the docker-compose configuration in a file called `stack.yml`. To run the configuration using `docker compose run docker-compose -f stack.yml up -d`.

## The Main Story  
### Using nginx  
My first attempt was to install [nginx](https://nginx.org) (not a Docker container) as a reverse proxy and pointed it to my container with a fairly basic config:
```nginx
server {
  listen 80;
  server_name example.com;
  location / {
     proxy_pass http://localhost:8080; # 8080 being the exposed port on the WordPress container
     proxy_redirect     off;
     proxy_set_header   Host $host;
     proxy_set_header   X-Real-IP $remote_addr;
     proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
     proxy_set_header   X-Forwarded-Host $server_name;
  }
}
```

```yml
version '3.3'
services:
  wordpress:
    depends_on:
      - db
    image: wordpress:latest
    restart: always
    ports:
      - "8080:80"
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_USER: <username>
      WORDPRESS_DB_PASSWORD: <password>

db:
  image: mysql:5.7
  volumes:
    - db_data:/var/lib/mysql
  restart: always
  environment:
    MYSQL_ROOT_PASSWORD: <root_password>
    MYSQL_DATABASE: <database_name>
    MYSQL_USER: <username>
    MYSQL_PASSWORD: <password>

volumes:
  db_data:
```

Which (magically) worked!
Yay me.
Seeing as this works, adding HTTPS using [LetsEncrypt](https://letsencrypt.org) and [Certbot](https://certbot.eff.org) surely won’t be an issue, right?
Right?
Wrong.

After following the Certbot instructions, which consist of installing Certbot and running it, I kept getting 502 Bad Gateway errors when trying to access my WordPress setup.
Nothing I did with the nginx config seemed to address the issue, so I finally moved on to trying to set up nginx using the [nginx docker container](https://hub.docker.com/_/nginx/).
While setting that up I found a useful guide on [The Polyglot Developer](https://www.thepolyglotdeveloper.com/2017/03/nginx-reverse-proxy-containerized-docker-applications/) that provided some insight into how to reverse proxy between containers using the nginx container.

### Using the nginx docker container  
Setting up the nginx Docker container went well, and using just HTTP it worked!
At this point the configuration begun to get a little more complicated.

After playing around with the nginx container, I realized it would be a hassle to build my own container that would include Certbot so I could generate a certificate, so I used the [really/certbot-nginx](https://hub.docker.com/r/really/nginx-certbot/) container.
This container has both nginx and Certbot installed, configured and ready to use, so all I had to was copy over the nginx configuration.

I found that the best way to do that was to create a volume in the `/etc/nginx/conf.d` directory of the container, and move my nginx configuration to the corresponding location (`/data/nginx/conf.d/`, see `stack.yml` below)

```nginx
upstream docker-wordpress {
  server wordpress; # server should be the same as the wordpress service in 'stack.yml', see https://docs.docker.com/network/bridge/
}

server {
  listen 80;
  server_name example.com;
  location / {
    proxy_pass http://docker-wordpress;
    proxy_redirect     off;
    proxy_set_header   Host $host;
    proxy_set_header   X-Real-IP $remote_addr;
    proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header   X-Forwarded-Host $server_name;
  }
}
```

```yml
version '3.3'
services:
  reverseproxy:
    image: really/nginx-certbot
    ports:
      - "80:80"
    volumes:
      - /data/nginx/conf.d:/etc/nginx/conf.d:rw
      - /data/letsencrypt:/etc/letsencrypt:rw

  wordpress:
    depends_on:
      - reverseproxy
      - db
    image: wordpress:latest
    restart: always
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_USER: <username>
      WORDPRESS_DB_PASSWORD: <password>

  db:
    image: mysql:5.7
    volumes:
      - db_data:/var/lib/mysql
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: <root_password>
      MYSQL_DATABASE: <database_name>
      MYSQL_USER: <username>
      MYSQL_PASSWORD: <password>

volumes:
  db_data:
```

For this configuration to work over HTTPS, I had to make a slight modification to the configuration, and manually run Certbot to get the certificate.
The change in the configuration is minor; add “443” to the exposed ports for `reverseproxy` in `stack.yml:` `"- 443:443"`. Not a very big change.

```yml
version '3.3'
services:
  reverseproxy:
    image: really/nginx-certbot
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /data/nginx/conf.d:/etc/nginx/conf.d:rw
      - /data/letsencrypt:/etc/letsencrypt:rw

  wordpress:
    depends_on:
      - reverseproxy
      - db
    image: wordpress:latest
    restart: always
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_USER: <username>
      WORDPRESS_DB_PASSWORD: <password>

  db:
    image: mysql:5.7
    volumes:
      - db_data:/var/lib/mysql
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: <root_password>
      MYSQL_DATABASE: <database_name>
      MYSQL_USER: <username>
      MYSQL_PASSWORD: <password>

volumes:
  db_data:
```

To generate the certificate, I logged into the docker container using `docker exec -it <name_of_reverseproxy_docker_image> /bin/sh` (you can find it the name of your running images by running `docker ps`).

Once logged into the image I ran `certbot --nginx -d example.com` and went through the prompts.
All went well; I had the certificates, Certbot modified the nginx config for me, I could open the WordPress website, and everything was good.

Except it wasn’t loading the static content like CSS and JavaScript.
Ugh.

### Introducing nginx-proxy  
I figured that the issue with the static content had to be because the reverse proxy wasn’t properly set up, and I wasn’t sure how to proxy the data correctly, so I started looking into other solutions.
Eventually I found out about [jwilder/nginx-proxy](https://hub.docker.com/r/jwilder/nginx-proxy/).
It allows you to configure an nginx Docker container as a reverse proxy, _using configuration in your Docker Compose file_.
Brilliant!

This solves the problem of me having to worry about the proxy configuration, and it’s been shown to work with other configurations!

What makes this even better is that there’s a companion container called [letsencrypt-nginx-proxy-companion](https://hub.docker.com/r/jrcs/letsencrypt-nginx-proxy-companion/) that automates LetsEncrypt certificate generation! All you have to do is configure your containers correctly.
This removes the need for an nginx config, and removes the need of manually entering the container to generate the certificate.

The `stack.yml` now looks like this:
```yml
version: '3.3'
services:
  nginx-proxy:
    image: jwilder/nginx-proxy
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - nginx-certs:/etc/nginx/certs:ro
      - nginx-vhost:/etc/nginx/vhost.d
      - nginx-html:/usr/share/nginx/html
      - /var/run/docker.sock:/tmp/docker.sock:ro
    labels:
      - com.github.jrcs.letsencrypt_nginx_proxy_companion.nginx_proxy

  nginx-proxy-letsencrypt:
    image: jrcs/letsencrypt-nginx-proxy-companion
    depends_on:
        - nginx-proxy
    volumes:
      - nginx-certs:/etc/nginx/certs:rw
      - nginx-vhost:/etc/nginx/vhost.d
      - nginx-html:/usr/share/nginx/html
      - /var/run/docker.sock:/var/run/docker.sock:ro

  wordpress:
    depends_on:
      - db
      - nginx-proxy
      - nginx-proxy-letsencrypt
    image: wordpress:latest
    restart: always
    ports:
      - "8080:80"
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_USER: <username>
      WORDPRESS_DB_PASSWORD: <password>
      VIRTUAL_HOST: example.com
      LETSENCRYPT_HOST: example.com
      LETSENCRYPT_EMAIL: admin@example.com

  db:
    image: mysql:5.7
    volumes:
      - db_data:/var/lib/mysql
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: <root_password>
      MYSQL_DATABASE: <database_name>
      MYSQL_USER: <username>
      MYSQL_PASSWORD: <password>

volumes:
  db_data:
  nginx-certs:
  nginx-vhost:
  nginx-html:
```

This doesn’t seem to be working quite yet, and I’m not sure why.
Maybe the initial image has to be configured like this?
I don’t really know why this doesn’t work.

## Update August 15, 2018
What was happening is that every time I opened up the page I just got a white screen, but everything else seemed to be working.
It wasn’t until I had tried this several times that I realized that what was missing was the theme I had set on the page previously.
At some point I had closed the container and the theme configuration was deleted.
This meant that the _site_ wouldn’t load, but the admin site _would_.

To fix the issue all I had to do was open up the admin portal at `https://example.com/wp-admin` and re-install the theme, and bingo!
Everything working as intended, HTTPS and all!