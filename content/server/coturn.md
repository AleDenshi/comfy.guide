---
title: Coturn
date: 2023-01-01
description: 'A STUN and TURN server that allows users to perform WebRTC calls while being behind NATs.'
icon: webrtc.svg
ports: [3478, 5349]

## Author Information
author: Denshi
---

[Coturn](https://github.com/coturn/coturn) is a libre **STUN** and **TURN** server software that allows users of chat protcols (Such as XMPP and Matrix to perform WebRTC **voice and video calls** despite them being behind NATs.

Almost every self-hosted voice and video conferencing program (such as Nextcloud's Talk app) will **require** Coturn or some other equivalent turnserver to function properly.

### Note on ejabberd

If you’re installing [ejabberd](/server/ejabberd), then you don’t need Coturn. Ejabberd comes with a TURN server built-in, and you should only setup ejabberd to connect to Coturn if you intend on running multiple chat services like Matrix and XMPP.

## Installation
Coturn is available in the Debian repositories:

```sh
sudo apt install coturn
```

### Base configuration
Coturn's configuration file is `/etc/turnserver.conf`. There are a few aspects that need to be changed in order to get a fully-functioning turnserver.

Here is an example of some sane defaults:

```sh
server-name={{<hl>}}turn.example.org{{</hl>}}
realm={{<hl>}}turn.example.org{{</hl>}}
listening-ip={{<hl>}}your_public_ip{{</hl>}}

listening-port=3478
min-port=10000
max-port=20000

# The "verbose" option is useful for debugging issues
verbose
```

### Authentication
There are two options for authentication on a turnserver:

* **Usernames** and **passwords,**
* or **authentication secrets.**

Depending on what self-hosted service is being used in conjunction with Coturn, you may need one or the other of these two options.

#### Usernames and Passwords

To utilize username and password authentication with Coturn, add the following configuration in `turnserver.conf`:

```sh
lt-cred-mech
user=username:password
```

#### Authentication Secrets

To utilize authentication secrets with Coturn, add the following configuration in `turnserver.conf`:

```sh
use-auth-secret
static-auth-secret=your_auth_secret
```

### TURNS (TLS Encryption)
Some self-hosted services (such as Matrix and XMPP) may support the use of **TURNS:** An encrypted version of TURN, which allows for WebRTC connections to be established with the use of an encrypted TLS tunnel, just like HTTPS allows for encrypted viewing of websites.

To utilize TURNS, certificates need to be declared for **turn.example.org** in `turnserver.conf`:

```sh
cert=/etc/certs/turn.example.org/fullchain.pem
pkey=/etc/certs/turn.example.org/privkey.pem
```

This is assuming that the certs are located in `/etc/certs/DOMAIN` and all owned by the `turnserver` user.
### Starting Coturn
After all configuration changes are complete, Coturn can be started with its systemd daemon:

```sh
sudo systemctl restart coturn
```

Congratulations! You've successfully setup a Coturn server!

