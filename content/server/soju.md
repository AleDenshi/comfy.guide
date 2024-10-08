---
## Guide Information
title: "Soju"
description: A user-friendly IRC bouncer
icon:
date: 2024-06-26
ports: [6697]

## Author Information
author: Ángel Castañeda
email: angel@acsq.me
matrix: "@acsquared:cyberia.club"

---

Soju is an IRC bouncer that turns the protocol into something modern everyone can use with chat history, persistent connections, and multi-server support.

This guide assumes you have a subdomain at `irc.example.org` and are port forwarding at 6697.

## Installation

Soju is not in Debian 12's repos, so we'll need to build it from source.

```sh
VERSION=v0.8.0
curl -fLO https://codeberg.org/emersion/soju/archive/$VERSION.tar.gz
tar xvf $VERSION.tar.gz
rm $VERSION.tar.gz
mv soju-$VERSION soju
```

Soju's latest release depends on Go 1.19 which happens to be the exact same version in Debian's repos. Let's install some more dependencies to build our binaries:

```sh
apt install golang scdoc libsqlite3-dev
export GOFLAGS="-tags=libsqlite3"
make
make install
```

## Deployment

### Unix User

From the docs:
> soju is designed to be run as a system-wide service under a separate user account.

Let's make that user:

```sh
useradd -m -d /var/lib/soju soju
```

### SSL certificates

```sh
apt install certbot
certbot certonly -d {{<hl>}}irc.example.org{{</hl>}} --register-unsafely-without-email
chmod 0755 /etc/letsencrypt/{live,archive}
chmod 0640 /etc/letsencrypt/live/{{<hl>}}irc.example.org{{</hl>}}/privkey.pem
chgrp soju /etc/letsencrypt/live/{{<hl>}}irc.example.org{{</hl>}}/privkey.pem
```

Now let's automate soju restarting each time we pull a new certificate:

Write out this script to /etc/letsencrypt/renewal-hooks/post/soju.sh:

```sh
#!/bin/sh -eu
systemctl reload soju
```

Then make it executable:

```sh
chmod +x /etc/letsencrypt/renewal-hooks/post/soju.sh
```

## Configuration

### Soju

Now let's fill out a configuration file at /etc/soju/config:

```scfg
listen        ircs://
# optional: add unix socket to control soju through the cli
listen        unix+admin://

tls           /etc/letsencrypt/live/{{<hl>}}irc.example.org{{</hl>}}/fullchain.pem /etc/letsencrypt/live/{{<hl>}}irc.example.org{{</hl>}}/privkey.pem
db            sqlite3 /var/lib/soju/main.db
message-store db
hostname      {{<hl>}}irc.example.org{{</hl>}}
auth          internal
title         "anon's soju bouncer"
```

Make an admin user called anon:

```sh
sojudb create-user anon -admin
```

### systemd service

Now let's make a systemd service that will start soju automatically for us. Write this to /etc/systemd/system/soju.service:

```systemd
[Unit]
Description=soju IRC bouncer service
Documentation=https://soju.im/
Documentation=man:soju(1) man:sojuctl(1)
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
User=soju
Group=soju
DynamicUser=yes
StateDirectory=soju
ConfigurationDirectory=soju
RuntimeDirectory=soju
AmbientCapabilities=CAP_NET_BIND_SERVICE
ExecStart=/usr/local/bin/soju
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Then enable our new service to start on boot, and start it:

```sh
systemctl daemon-reload
systemctl enable --now soju
```

## Usage

### Client Configurations

Soju can be used by most regular IRC clients, but clients that support the soju.im/bouncer-networks extension make multiple servers work seamlessly.

Here is a table of recommended clients that support this extension.

| Client                                                      | Interface    |
| :---------------------------------------------------------- | :----------  |
| [Senpai](https://sr.ht/~delthas/senpai/)                    | Terminal/TUI |
| [Goguma](https://f-droid.org/packages/fr.emersion.goguma/)  | Android/iOS  |
| [Gamja](https://sr.ht/~emersion/gamja/)                     | Web          |
| [Chathistorysync](https://sr.ht/~emersion/chathistorysync/) | Plain Text   |

Configurations for clients not supporting the extension can be found [here](https://codeberg.org/emersion/soju/src/branch/master/contrib/clients.md).

Here is an example client configuration for senpai.

```scfg
address  {{<hl>}}irc.example.org{{</hl>}}
nickname anon
password <password for anon>
```

### New Networks

Several of the clients above have special user interfaces to add new servers, but the default way to do it with any client is with the `network create` command.

For example, this is how we'd add [libera.chat](https://libera.chat) to our server:

```
/msg BouncerServ network create -addr irc.libera.chat -name libera -realname "mr anon"
```

### New Users

Your bouncer can be used by friends too. To set up a user, an admin user can run the following:

```
/msg BouncerServ user create <username> -password <password>
```

Note: If you set up the `unix+admin://` listen directive in your config, you can run any of the previous commands with `sojuctl <BouncerServ cmd>` in a shell.

## Further Configuration

You should have a usable bouncer by now, but there are more config options worth looking into. Read the documentation in soju's man page (`man soju`) or take a look at the [contrib](https://codeberg.org/emersion/soju/src/branch/master/contrib/) section of its Git repository.

Of note:

* websocket listeners to configure a local gamja server

* file-upload protocol and drivers

* motd if you wanna show off some cool [ascii art](https://www.asciiart.eu/computers/linux)
