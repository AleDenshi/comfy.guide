---
## Software Information
title: Matrix Synapse
date: 2023-01-29
description: Host your own federated messaging server using the Matrix protocol.
icon: matrix.svg
ports: [80, 443, 8448]

## Author Information
author: Denshi
---

[Matrix](https://matrix.org) is a free/libre API and protocol for chat and voice communication; Think DIY Discord, except without all the spyware. One can setup their own Matrix homeserver using either the [Synapse](https://github.com/matrix-org/synapse) or [Dendrite](https://github.com/matrix-org/dendrite) software; A Matrix server supports messaging, encrypted messaging, voice and video calls (once a turn server is set up), and many other services.

## Initial Server Setup
This section covers base installation and setup of Matrix-Synapse, about enough to get the homeserver running; If you wish to actually register an account to the instance and use it, consult the further server configuration part.

### Prerequisites
* Debian 10 or newer
* [NGINX](/server/nginx) installed
* Ports 80, 443 and 8448 opened
* Your own domain with an A DNS entry set to the public IPv4 address of your server
* Basic UNIX knowledge

### Webserver Configuration
Before you install any packages, there has to be an appropriate configuration for Matrix-Synapse in your `/etc/nginx/sites-available` directory. Here's an example configuration:

```nginx
server {
    server_name {{<hl>}}example.org{{</hl>}};

    listen 80;
    listen [::]:80;

    listen 443 ssl http2 default_server;
    listen [::]:443 ssl http2 default_server;
       
    listen 8448 ssl http2 default_server;
    listen [::]:8448 ssl http2 default_server; 
    
    location ~* ^(\/_matrix|\/_synapse|\/_client) {
        proxy_pass http://localhost:8008;
        proxy_set_header X-Forwarded-For $remote_addr;
        client_max_body_size {{<hl>}}50M{{</hl>}};
    }

    # These sections are required for client and federation discovery
    # (AKA: Client Well-Known URI)
    location /.well-known/matrix/client {
        return 200 '{"m.homeserver": {"base_url": "https://{{<hl>}}example.org{{</hl>}}}}';
        default_type application/json;
        add_header Access-Control-Allow-Origin *;
    }

    location /.well-known/matrix/server {
        return 200 '{"m.server": "{{<hl>}}example.org{{</hl>}}:443"}';
        default_type application/json;
        add_header Access-Control-Allow-Origin *;
    }
}
```

**(This example configuration file contains all configuration needed to utilize both federation and client Well-Known URI)**

This configuration file should be placed in `/etc/nginx/sites-available` and then symbolically linked to `/etc/nginx/sites-enabled`:

```sh
sudo ln -s /etc/nginx/sites-available/{{<hl>}}example.org{{</hl>}} /etc/nginx/sites-enabled
```

### Installing packages
To actually install Matrix-Synapse on Debian, first add the official Matrix Debian repo to your system:

```sh 
sudo apt install -y lsb-release wget apt-transport-https
sudo wget -O /usr/share/keyrings/matrix-org-archive-keyring.gpg https://packages.matrix.org/debian/matrix-org-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/matrix-org-archive-keyring.gpg] https://packages.matrix.org/debian/ $(lsb_release -cs) main" |
    sudo tee /etc/apt/sources.list.d/matrix-org.list
```

Then update the repository database and install the `matrix-synapse-py3` package:

```sh
sudo apt update
sudo apt install matrix-synapse-py3
```
Now simply reload the following services with `systemctl`:

```sh
sudo systemctl restart nginx matrix-synapse
```

### Encryption with Certbot

Begin by installing Certbot to your Debian system, along with the NGINX extension:

```sh
sudo apt install python3-certbot python3-certbot-nginx
```
As with any [NGINX](server/nginx) website, Let's Encrypt's Certbot tool can be used to not only generate an encryption certificate for your domain, but also write the appropriate configuration to the server entry in `/etc/nginx/sites-available`.

We can do all this with the following command:
```sh
 sudo certbot --nginx -d {{<hl>}}example.org{{</hl>}}
```

And you've done it! Matrix-Synapse is now successfully installed and secured!

## Further Server Configuration

While the basic setup above is enough for a private server, additional features like **voice and video calling, a faster PostgreSQL database** and **URL previews** can all be enabled through further configuration.

### Enabling PostgreSQL
It's highly recommended to use PostgreSQL as the database for your Matrix homeserver instead of the default SQLite database configuration; To set this up, begin by installing PostgreSQL:

```sh
apt install postgresql
```

Then start the daemon:

```sh
systemctl restart postgresql
```

Now create a user named `synapse_user` to manage your database:
```sh
su -c "createuser --pwprompt synapse_user" postgres
```

And finally, create the actual database:
```sh
su -c "psql -c 'CREATE DATABASE synapse ENCODING 'UTF8' LC_COLLATE='C' LC_CTYPE='C' template=template0 OWNER synapse_user;'" postgres
```

Now edit the database configuration in `/etc/matrix-synapse/homeserver.yaml` and comment out the following lines for the previous SQLite configuration: 

```yml
# database:
  # name: sqlite3
  # args:
    # database: DATADIR/homeserver.db
```

*Note: The example above is how yours should look like after it's commented out.*

Then, uncomment the following configuration above, and set the appropriate entries:
```yaml
database:
  name: psycopg2
  args:
    user: {{<hl>}}synapse_user{{</hl>}}
    password: {{<hl>}}secretpassword{{</hl>}}
    database: {{<hl>}}synapse{{</hl>}}
    host: localhost
    cp_min: 5
    cp_max: 10
```

## Enabling user registration

Enabling user registration means anyone will be able to register a Matrix account to your instance; This includes yourself, and enabling this feature is required to register an admin account. **Please note, Matrix accounts cannot be deleted, only "deactivated".**

Enabling registration is as simple as uncommenting and changing one line in the `/etc/matrix-synapse/homeserver.yaml` file:

```yaml
enable_registration: true
```

To be able to create users from your homeserver with the `register_new_matrix_user`  command, you also need to uncomment this line in your Matrix config:

```yaml
 registration_shared_secret: {{<hl>}}???{{</hl>}}
```
Then simply restart Matrix-Synapse with:

```sh
sudo systemctl restart matrix-synapse
```

Now, use the `register_new_matrix_user` command, which is installed alongside the `matrix-synapse-py3` package, to create a new admin user on your server:

```sh
sudo register_new_matrix_user -u {{<hl>}}username{{</hl>}} -p {{<hl>}}password{{</hl>}} -a -c /etc/matrix-synapse/homeserver.yaml
```

### Client Well-Known URI

An optional but recommended aspect to configure is the client Well-Known URI, which allows any user to simply type in their matrix username, for example `@denshi:denshi.org`, into any Matrix client, and have this automatically resolve their homeserver, skipping the homeserver setting step entirely;

This is already setup in the NGINX configuration previously stated in this guide.

### Federation

As mentioned before in the Initial Setup section of this page, the NGINX example configuration contains all the lines needed for federation to successfully work under it; No further configuration necessary!

You can test the federation by entering your server address into the [Matrix Federation Tester.](https://federationtester.matrix.org)

However, some extra features can be enabled to increase the usability of your homeserver over federation. In `homeserver.yaml`, the following lines can be edited:

```yaml
allow_public_rooms_over_federation: true
```

This can be un-commented to allow users to add your homserver to their list of servers (in a client like Element) and see a list of all the public rooms.

```yaml
allow_public_rooms_without_auth: true
```

This can be un-commented to enable guests to see public rooms without authenticating.


### Voice and Video Calls

For native voice and video call support, the Synapse homserver needs to interface with a working **TURN and STUN Server.**

First, follow the guide on installing and setting up [coturn](/server/coturn), setting either a shared secret or username-password pair for authentication.

Then, in `/etc/matrix-synapse/homeserver.yaml`, edit the configuration as follows:

```yaml
turn_uris: [ "turn:{{<hl>}}turn.example.org{{</hl>}}?transport=udp", "turn:{{<hl>}}turn.example.org{{</hl>}}?transport=tcp" ]

## This is how long call credentials are valid. Lessen to prevent abuse.
turn_user_lifetime: 86400000

## Keep this enabled unless for security reasons.
turn_allow_guests: True
```

If you're using a shared secret, add the following config:

```yaml
turn_shared_secret: "{{<hl>}}your secret here{{</hl>}}"
```

Otherwise, add this config if you're using username-password pairs:

```yaml
turn_username: "{{<hl>}}turnserver_username{{</hl>}}"
turn_password: "{{<hl>}}turnserver_password{{</hl>}}"
```

### URL Previews

Matrix-Synapse supports URL previews; To enable them, change this line to `true` in `/etc/matrix-synapse/homeserver.yaml`:

```yaml
url_preview_enabled: {{<hl>}}true{{</hl>}}
```

> Make sure to uncomment the `url_preview_ip_range_blacklist:` section; **Otherwise, Synapse will refuse to start up again!**
