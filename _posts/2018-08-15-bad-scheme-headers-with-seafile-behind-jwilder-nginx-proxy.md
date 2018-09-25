---
layout: post
title: Bad Scheme Headers with Seafile behind jwilder/nginx-proxy
---

## The problem  
After setting up [Seafile](https://seafile.com/) in a docker container following a guide in their documentation and configuring a URL and SSL using [jwilder/nginx-proxy](https://hub.docker.com/r/jwilder/nginx-proxy/) and [jrcs/letsencrypt-nginx-proxy-companion](https://hub.docker.com/r/jrcs/letsencrypt-nginx-proxy-companion/) I got a 400 Bad Request error when trying to open the URL.
In addition to it giving me this error it also said “Bad scheme headers”.

## The Solution  
Open the container and comment out the `proxy_set_header X-Forwarded-Ssl $proxy_x_forwarded_ssl;` line in `/etc/nginx/conf.d/default.conf`.

[Source](https://groups.google.com/forum/#!topic/nginx-proxy/T-B3HCpY1ug).
