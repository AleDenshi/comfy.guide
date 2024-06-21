---
## Guide Information
title: ejabberd
date: 2022-03-29
icon: ejabberd.png
description: A chat server based on XMPP.
ports: [5000, 5222, 5269, 5280, 5281]

## Author Information
author: Denshi
---

[Ejabberd](https://ejabberd.im) is a server for the XMPP protocol written in Erlang. It's more scalable, and easier to setup than [Prosody](/server/prosody) due to having most of its modules built-in and pre-configured by default.


## Subdomains

Ejabberd assumes that you have already created all the **required and optional subdomains** for its operation prior to running it.

Depending on the usecase, you may need any or all of the following domains for XMPP functionality:

-   **example.org** - Your XMPP hostname
-   **conference.example.org** - For `mod_muc`, Multi User Chats (MUCs)
-   **upload.example.org** - For `mod_http_upload`, file upload support
-   **proxy.example.org** - For `mod_proxy65`, SOCKS5 proxy support
-   **pubsub.example.org** - For `mod_pubsub`, publish-subscribe support (A fancier RSS)

Only the **example.org** domain is required for basic, private chat usage.
If you do **not** wish to use a certain domain, just disable it's associated module and ejabberd won't complain when it can't find it's associated certificate.
For example, if you don't want [Publish-Subscribe](https://xmpp.org/extensions/xep-0060.html) support, just comment out the `mod_pubsub` config in `/opt/ejabberd/ejabberd.yml`:

```yml
##  mod_pubsub:
##    access_createnode: pubsub_createnode
##    plugins:
##      - flat
##      - pep
##    force_node_config:
##      ## Avoid buggy clients to make their bookmarks public
##      storage:bookmarks:
##        access_model: whitelist
```

This guide will assume **all these subdomains** have been created.

### Custom Subdomains

If you wish to customize any of these domains, edit `/opt/ejabberd/conf/ejabberd.yml` and under every appropriate module that needs a subdomain, add the following setting:
```yml
mod_muc:
  host: {{<hl>}}muc.example.org{{</hl>}}
```

## Installation

To get the latest version of ejabberd, you need to first setup the ejabberd apt repositories:

```sh
curl -o /etc/apt/sources.list.d/ejabberd.list https://repo.process-one.net/ejabberd.list
curl -o /etc/apt/trusted.gpg.d/ejabberd.gpg https://repo.process-one.net/ejabberd.gpg
```

Then update the repositories and install the `ejabberd` package:

```sh
apt update
apt install ejabberd
```

## Configuration

The ejabberd server is configured in `/opt/ejabberd/conf/ejabberd/ejabberd.yml`.
Changes are only applied by restarting the ejabberd daemon in systemd:

```sh
systemctl restart ejabberd
```

### Hostnames

The **XMPP hostname** is specified in the `hosts` section of
`ejabberd.yml`:

```yml
hosts:
  - {{<hl>}}example.org{{</hl>}}
```

### Certificates

Unlike [Prosody,](https://prosody.im) ejabberd doesn\'t come equipped with a script that can automatically copy over the relevant certificates to a directory where the ejabberd user can read them.

One way of organizing certificates for ejabberd is to have them stored in `/opt/ejabberd/certs`, with each domain having a separate directory for both the fullchain cert and private key.

Before continuing, create the appropriate directory:

```sh
mkdir -p /opt/ejabberd/certs
```

Using certbot, this process can be easily automated with these commands:

```bash
DOMAIN={{<hl>}}example.org{{</hl>}}

# Set the domain names you want here
declare -a subdomains=("" "conference." "proxy." "pubsub." "upload.")

for i in "${subdomains[@]}"; do
    certbot --nginx -d $i$DOMAIN certonly
    mkdir -p /opt/ejabberd/certs/$i$DOMAIN
    cp /etc/letsencrypt/live/$i$DOMAIN/fullchain.pem /opt/ejabberd/certs/$i$DOMAIN/
    cp /etc/letsencrypt/live/$i$DOMAIN/privkey.pem /opt/ejabberd/certs/$i$DOMAIN/
done
```
*Note: Just like with Prosody, you might want to write this script to a file and setup a cronjob to run it periodically. This should help prevent your certificates from expiring.*

Make sure all the certificates are readable by the `ejabberd` user:
```sh
chown -R ejabberd:ejabberd /opt/ejabberd/certs
```

To enable the use of all these certificates in ejabberd, the following
configuration is necessary:

```yml
certfiles:
  - "/opt/ejabberd/certs/*/*.pem"
```

### Admin User

The **admin user** can be specified in `ejabberd.yml` under the `acl` section:

```yml
acl:
  admin:
    user: admin
```

This would make **admin@example.org** the user with administrator privileges.

### File Uploads

To ensure full compliance with XMPP standards, add the following configuration to `mod_http_upload`:

```yaml
mod_http_upload:
    put_url: https://@HOST@:5443/upload
    docroot: {{<hl>}}/var/www/upload{{</hl>}}
    custom_headers:
      "Access-Control-Allow-Origin": "https://@HOST@"
      "Access-Control-Allow-Methods": "GET,HEAD,PUT,OPTIONS"
      "Access-Control-Allow-Headers": "Content-Type"
```

Make sure to create and give the `ejabberd` user ownership of `/var/www/upload` or any other directory you choose to use for file uploads:

```sh
chown -R ejabberd:ejabberd /var/www/upload
```

### Message Archives

The ejabberd server supports keeping archives of messages through its `mod_mam` module. This can be enabled by uncommenting the following lines:

```yml
mod_mam:
  assume_mam_usage: true
  default: always
```

## Database

### Why use a database?

We can find the following comment in the `mod_mam` section of `ejabberd.yml`:

```yml
mod_mam:
  ## Mnesia is limited to 2GB, better to use an SQL backend
  ## For small servers SQLite is a good fit and is very easy
  ## to configure. Uncomment this when you have SQL configured:
  ## db_type: sql
```

As these comments imply, an **[SQL backend](/server/sql)** is strongly recommended if you wish to use your ejabberd server for anything more than just testing. Ejabberd supports **MySQL, SQLite** and **PostgreSQL.** For the purpose of efficiency, this guide will use **PostgresSQL** because other server software like [Matrix](/server/matrix) and [PeerTube](/server/peertube) support it.

### Installing PostgreSQL

PostgreSQL is available in the Debian repositories:

```sh
apt install postgresql
```

In addition, you will have to install the **appropriate headers for Erlang,** the language ejabberd is written in, so it can actually interact with the PostgreSQL server:
```sh
apt install erlang-p1-pgsql
```

Start the PostgreSQL daemon to begin using it:

```sh
systemctl start postgresql
```

### Creating the Database

To create the database, first create a PostgreSQL user for ejabberd:

```sh
su -c "createuser --pwprompt ejabberd" postgres
```

Then, create the database and make `ejabberd` its owner:

```sh
su -c "psql -c 'CREATE DATABASE ejabberd OWNER ejabberd;'" postgres
```

### Importing Database Scheme

Ejabberd does **not** create the database scheme by default; It has to be imported into the database before use.

```sh
su -c "curl -s https://raw.githubusercontent.com/processone/ejabberd/master/sql/pg.sql | psql ejabberd" postgres
```

### Configuring ejabberd to use PostgreSQL

Finally, add the following configuration to `ejabberd.yml`:

```yml
sql_type: pgsql
sql_server: "localhost"
sql_database: "{{<hl>}}ejabberd{{</hl>}}"
sql_username: "{{<hl>}}ejabberd{{</hl>}}"
sql_password: "{{<hl>}}psql_password{{</hl>}}"

default_db: sql
```
That line at the end sets **every module's database** to default to the `sql` backend. This includes the `mod_mam` module, so all our data is being stored with PostgreSQL.

### Voice and Video Calls

Ejabberd supports the **TURN** and **STUN** protocols to allow internet users behind NATs to perform voice and video calls with other XMPP users. **This is enabled by default using [ejabberd_stun](https://docs.ejabberd.im/admin/configuration/listen#ejabberd-stun-1).**

**However,** if you plan on running ejabberd alongside **other applications** that require TURN and STUN, such as Matrix, then you'll have to setup your own external TURN server using Coturn.

#### Setup with Coturn and `mod_stun_disco`

Firstly, setup a TURN and STUN server with [Coturn,](/server/coturn) using an **authentication secret.**

Then, edit `mod_stun_disco` to contain the appropriate information for
your turnserver:

```yml
  mod_stun_disco:
    secret: "{{<hl>}}your_auth_secret{{</hl>}}"
    services:
      -
        host: {{<hl>}}turn.example.org{{</hl>}}
        type: stun
      -
        host: {{<hl>}}turn.example.org{{</hl>}}
        type: turn
```


## Using ejabberd

### Registering the Admin User

To begin using ejabberd, firstly start the ejabberd daemon:

```sh
systemctl restart ejabberd
```

Then, using `ejabberdctl` as the ejabberd user, register the admin user which is set in `ejabberd.yml`:

```sh
su -c "ejabberdctl register {{<hl>}}admin example.org password{{</hl>}}" ejabberd
```

This will create the user **admin@example.org.**

### Using the Web Interface

By default, ejabberd has a web interface accessible from **http://example.org:5280/admin**. When accessing this interface, you will be prompted for the admin credentials:

![](ejabberd-login.webp)

After signing in with the admin credentials, you will be able to manage your ejabberd server from this web interface:

![](ejabberd-admin.webp)

## Further Configuration

For a deeper look into all the modules and options, have a look at the following ejabberd documentation:
- Ejabberd's [Listen Modules](https://docs.ejabberd.im/admin/configuration/listen/) and [Listen Options](https://docs.ejabberd.im/admin/configuration/listen-options/)
- Ejabberd's [Top-Level Options](https://docs.ejabberd.im/admin/configuration/toplevel/)
- Ejabberd's [Modules' Options](https://docs.ejabberd.im/admin/configuration/modules/)

*And with that, you've successfully setup your ejabberd XMPP server!*
