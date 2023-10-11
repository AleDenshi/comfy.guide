---
## Software Information
title: NGINX
date: 2023-01-29
description: Host static websites, services over HTTP(S) and more.
icon: nginx.svg
ports: [80, 443]

## Author Information
author: Denshi
---

NGINX (pronounced "Engine-X") is a free/libre webserver and reverse-proxy software. It's basically what you're meant to be using instead of Apache2.

## Installation

NGINX is included in the Debian repositories:
```sh
sudo apt install nginx
```

## Configuration
By default, NGINX on Debian scans the `/etc/nginx/sites-enabled/` directory for webserver configuration files. The instruction to do so is included in the `/etc/nginx/nginx.conf` file:
```nginx
include /etc/nginx/modules-enabled/*.conf;
```

It is recommended to place server configuration files in the `/etc/nginx/sites-available` directory, and then **symbolically link** them to `/etc/nginx/sites-enabled` to let NGINX see the configurations:

```sh
# Enabling a configuration file:
ln -s /etc/nginx/sites-available/{{<hl>}}example.org{{</hl>}} /etc/nginx/sites-enabled/
# Disabling a configuration file:
unlink /etc/nginx/sites-enabled/{{<hl>}}example.org{{</hl>}}
```

> **Note:** While every example in this guide will have the configuration file named after the specific site it's configuring (`/etc/nginx/sites-enabled/example.org` for example.org, and so on) this is not a requirement for running a website. NGINX will scan any and all files included in `/etc/sites-enabled`, and the filename does not affect the behavior of NGINX.

The actual configuration file at `/etc/nginx/sites-available/{{<hl>}}example.org{{</hl>}}` to serve a static HTML page should look like this:

```nginx
server {
    listen 80;
    listen [::]:80;
    
    server_name {{<hl>}}example.org{{</hl>}};

    root /var/www/{{<hl>}}example.org{{</hl>}};
    index index.html;
    
    location / {
        try_files $uri $uri/ =404;
    }
}
```

The configuration above serves files from the `/var/www/example.org` directory. This can be set to any directory accessible by the `nginx` user.

## Encryption with Let's Encrypt
By using Let's Encrypt's `certbot` tool along with the `certbot-nginx` extension, one can enable encrypted traffic to their server and generate a full-chain encryption certificate.

Begin by installing Certbot and the NGINX extension for it:
```sh
apt install python3-certbot python3-certbot-nginx
```

Then, use the following command to do the rest:
```sh
certbot --nginx -d {{<hl>}}example.org{{</hl>}} --register-unsafely-without-email
```

Once certificate generation is complete, this command will bring up a prompt to either disable or enable redirection of non-encrypted traffic through the encrypted port. 
**It is recommended to enable Redirect.**

### Systemd Service
The `nginx` package on Debian includes a systemd service:
`sudo systemctl restart nginx`

**Once restarted, NGINX should find your configuration file at `/etc/nginx/sites-enabled/` and successfully serve your static HTML site!**

## Further Configuration
NGINX config files can be edited to add various functionality to a website. 
These options allow for NGINX to act as a powerful tool for much more than just serving static content.

### Enabling File View/Indexing
While Apache2 has this feature enabled by default, file indexing is turned off in NGINX unless the user specifies otherwise. Enabling auto-indexing requires for a location to be set, from where the files will be served:

```nginx
    location / {
        autoindex on;
    }
```
### Proxying
NGINX can proxy traffic from any network location and serve it over the ports specified in a config. To proxy traffic, one must specify the location where the traffic is to be served, and the originating address of the traffic:
```nginx
    location / {
        proxy_pass {{<hl>}}http://127.0.0.1:8008{{</hl>}};
    }
```

### Redirects
You can redirect any URL in NGINX to any other URL:
```nginx
server {
...
    rewrite ^/{{<hl>}}test{{</hl>}}$ {{<hl>}}https://test.example.org{{</hl>}} permanent;
...
}
```
This way, `https://example.org/test` redirects to `https://test.example.org`.

### Server Tokens
For security reasons, you might want to hide specific NGINX version information from being served. This can be done by uncommenting the following line in `/etc/nginx/nginx.conf`:
```nginx
server_tokens off;
```
