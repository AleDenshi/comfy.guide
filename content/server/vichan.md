---
# Guide Information
title: Vichan
date: 2023-01-01
icon: vichan.png
description: A simple imageboard engine written in PHP.
ports: [80, 443]

# Author Information
author: Denshi
---

[Vichan](https://github.com/vichan-devel/vichan/) is libre imageboard software written in PHP. It's a simple, minimalist social media solution, similar to popular sites like 4chan.

## Initial Setup
This guide runs you through the initial setup process for Vichan. This is a very barebones install, with no index page. The additional functionality seen on most popular board sites can be replicated utilizing "Themes".

Here are the additional packages required for Vichan to function properly:

```sh
apt install php-bcmath php-gd php-pdo php-mbstring php-mysql mariadb-server composer
```

### Installing Vichan with Composer
Vichan can be cloned from Github to a directory, where it will be served from:

```sh
git clone https://github.com/vichan-devel/vichan.git /var/www/{{<hl>}}chan.example.org{{</hl>}}
```
You can then utilize `composer` to install the PHP scripts needed by Vichan:

```sh
cd /var/www/{{<hl>}}chan.example.org{{</hl>}}
composer install
```
### Database setup
Vichan requires a MySQL database to function properly. This can be easily setup by running the following commands:

Enter the MySQL prompt:

```sh
mysql
```

Create the Vichan user...

```sql
CREATE USER 'vichan'@'localhost' IDENTIFIED BY '{{<hl>}}your_password{{</hl>}}';
```

Create the Vichan database, and grant the Vichan user the permissions...

```sql
CREATE DATABASE vichan_db;
use vichan_db;
GRANT ALL ON vichan_db.* TO 'vichan'@'localhost';
```

**Make sure to remember the Vichan user's username and password!**

### Webserver setup
For Vichan to be served over a specific domain (with TLS), a specific LEMP configuration is required. This guide will utilize NGINX:

```nginx
server {
        listen 80;
        listen [::]:80;

        root /var/www/{{<hl>}}chan.example.org{{</hl>}};
        index index.html;
        server_name {{<hl>}}chan.example.org{{</hl>}};
 
        location / {
            try_files $uri $uri/ =404;
        }
 
        location ~ \.php$ {
            include snippets/fastcgi-php.conf;
            fastcgi_pass unix:/var/run/php/php7.4-fpm.sock;
        }
}
```

#### HTTPS with Certbot
You can then utilize the `certbot` tool to get and setup the appropriate TLS certificate:

```sh
certbot --nginx -d {{<hl>}}chan.example.org{{</hl>}}
```

**This will automatically reconfigure your site's NGINX file to use port 443 with HTTPS.**

### Web setup
The recommended way of setting up Vichan, similarly to MediaWiki, is through the web installer.

Firstly, ensure that the PHP FastCGI Process Manager is started:

```sh
systemctl start php-fpm
```
Then, simply navigate to `https://{{<hl>}}chan.example.org{{</hl>}}/install.php` and go through the required setup steps. Make sure to enter the correct database information, including the MySQL username, password and database you plan to use for Vichan.

Congratulations! You've performed a base install of Vichan!
