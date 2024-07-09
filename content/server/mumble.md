---
## Software Information
title: Mumble
date: 2023-10-09
description: Voice call with friends.
icon: mumble.svg
ports: [64738]

## Author Information
author: Denshi
---

[Mumble](https://www.mumble.info/) is a light and high-quality libre voice chat protocol. You can self-host your own Mumble server and control your conversations.

## Installation

The Mumble server software, "murmur", is available in the Debian repositories:
```sh
apt install mumble-server
```

## Configuration

The Mumble server is configured in `/etc/mumble-server.ini`. Here you can customize information, like the welcome message:

```ini
; Welcome message, in HTML, shown to users when they join
welcometext="<b>Hello there, and welcome to my <i>Mumble server!</i></b>"
```

If you wish to run Mumble on a different port rather than the default, this can also be changed:
```ini
port=64738
```

In addition, you can also customize the number of users that can connect to your server in total:
```ini
users=30
```

### Certificate

If you plan on running your mumble server under a specific domain, like {{<hl>}}example.org{{</hl>}}, you can utilize a previously-obtained certificate (like the one used for [NGINX](/server/nginx) with certbot) to encrypt connections on your server with a valid TLS name. **Not including this doesn't make Mumble unencrypted, but it does ease the process for anyone running their server under a specific domain as users won't get a "failed certification" error.**

Begin by copying existing certificates to a location where the `mumble-user` can access them:

```sh
mkdir -p /usr/share/certs/{{<hl>}}example.org{{</hl>}}
cp /etc/letsencrypt/live/{{<hl>}}example.org{{</hl>}}/fullchain.pem /usr/share/certs/{{<hl>}}example.org{{</hl>}}/
cp /etc/letsencrypt/live/{{<hl>}}example.org{{</hl>}}/privkey.pem /usr/share/certs/{{<hl>}}example.org{{</hl>}}/
```

```ini
sslCert=/usr/share/certs/{{<hl>}}example.org{{</hl>}}/fullchain.pem
sslKey=/usr/share/certs/{{<hl>}}example.org{{</hl>}}/privkey.pem
```

Make sure to give appropriate file permissions to `mumble-user`:

```sh
chown -R mumble-user:mumble-user /usr/share/certs/{{<hl>}}example.org{{</hl>}}
```

### Public Directory

If you wish, you may publish your Mumble server to the public server directory accessible in the default Mumble client:
```ini
registerName={{<hl>}}MyMumble{{</hl>}}
registerPassword={{<hl>}}secret_password{{</hl>}}
registerUrl=http://www.mumble.info/
registerHostname={{<hl>}}example.org{{</hl>}}
registerLocation={{<hl>}}US{{</hl>}}
```

### Running as a Service

Simply restart the systemd service for Murmur to start the server:
```sh
systemctl restart mumble-server
```

And with that, you've done it! You're running your very own Mumble server.

## Using your Mumble Server

Once you've setup your Mumble server, you can use it by connecting to its IP address (or domain name pointing to the same server) through any Mumble client.

![The interface for setting up the certificate.](1-cert.png)

> **Important Note:** When starting Mumble for the first time, you will be asked to create a *certificate.* Do not lose this certificate, as this is practically your Mumble "identity". If you register a custom user as an admin on your server, you'll need the corresponding certificate to gain the administrative privileges. Once again, **don't lose it!**


### Registering as an Admin

By default, any Mumble user named `SuperUser` connecting to your server will have administator privileges. To set the special admin password used by this user, simply run the following on your server:

```sh
murmurd -supw {{<hl>}}super_secret_password{{</hl>}}
```

If you want to make a different user an admin, the steps are slightly more complicated. Begin by connecting as a regular user to your server:

![Connecting to the example Mumble server.](1-connecting.png)

**Right-click on your user, and select "register".** You will then be met with the following prompt, where you should click "Yes":

![Click yes at the prompt to register yourself](2-register.png)

**Now disconnect from the server** and change your username in the server's settings, setting it this time to `SuperUser`. You will be met with a prompt for a password, where you should enter the password you already set with the `murmurd -supw` command:

![Change your username to SuperUser and set the password correctly.](3-superuser.png)

Now, as the SuperUser, **right-click on your channel name and click "Edit..."** This will automatically open the "Groups" dialog, where you need to select the **admin** group from the drop-down menu.

![groups](4-groups.png)

After selecting the correct group from the drop-down menu, simply add your previously registered username to the admin group.

![Type in the user's name.](5-adduser.png)

You've successfully added your custom username to the admin group!

![Type in the user's name.](6-done.png)

