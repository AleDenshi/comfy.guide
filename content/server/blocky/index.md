---
## Guide Information
title: Blocky DNS
date: 2024-07-12
icon: blocky.svg
description: A secure DNS server that blocks ads.
ports: [53, 443, 843]

## Author Information
author: Denshi
---

[Blocky](https://0xerr0r.github.io/blocky/latest/) is a libre DNS server that, in addition to the standard DNS protocol, supports DNS-over-HTTPS and DNS-over-TLS. It also supports auto-updating ad blocking DNS lists in addition to your static `/etc/hosts` file. 

> DNS-over-TLS is the technology required by Android 9+ to set a **universal** DNS server. With this consideration, Blocky becomes an invaluable tool for blocking ads and [unwanted content](https://denshi.org/antiporn) on any smartphone, especially without root access.

## Installation

Begin by creating a dedicated directory for blocky and changing to it:

```sh
mkdir -p /opt/blocky
cd /opt/blocky
```

Download the latest version of Blocky from the [releases page](https://github.com/0xERR0R/blocky/releases/latest):

```sh
baseurl="https://github.com/0xERR0R/blocky/releases"
ver=$(basename "$(curl -w "%{url_effective}\n" -I -L -s -S $baseurl/latest -o /dev/null)")
curl -fLO "https://github.com/0xERR0R/blocky/releases/download/${ver}/blocky_${ver}_Linux_x86_64.tar.gz"

tar xvf blocky_*
```

## Configuration

Create a new file in `/opt/blocky` named `config.yaml` and place the following configuration in it:

```yml
# Upstream DNS server configuration
upstream:
  default:
    # Modify these default DNS servers to your liking
    - {{<hl>}}9.9.9.9{{</hl>}}
    - {{<hl>}}tcp-tls:fdns1.dismail.de:853{{</hl>}}
    - {{<hl>}}https://dns.digitale-gesellschaft.ch/dns-query{{</hl>}}

# Ports configuration
ports:
  dns: 53
  http: 4000
  tls: 853

# Logging, set this to "error" to avoid collecting user info
log:
  level: info
```

### systemd Service

To run blocky as a systemd service, begin by creating a `blocky` user to run the service, giving it permissions over the `/opt/blocky` directory:

```sh
useradd -d /opt/blocky blocky
```

add the following in `/etc/systemd/system/blocky.service`:

```systemd
[Unit]
Description=Blocky DNS
After=syslog.target
After=network.target

[Service]
RestartSec=2s
Type=simple
User=blocky
Group=blocky
WorkingDirectory=/opt/blocky/
ExecStart=/opt/blocky/blocky -c config.yaml
Restart=always

[Install]
WantedBy=multi-user.target
```


### DNS-over-HTTPS

To use DNS-over-HTTPS, begin by setting up a domain such as {{<hl>}}dns.example.org{{</hl>}}, and create an [NGINX configuration](/server/nginx) for it in `/etc/nginx/sites-enabled/{{<hl>}}dns.example.org{{</hl>}}`:

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name {{<hl>}}dns.example.org{{</hl>}};

    location / {
        proxy_pass http://127.0.0.1:4000;
    }
}
```

Use certbot to automatically download and configure your TLS certificate:

```sh
certbot --nginx -d {{<hl>}}dns.example.org{{</hl>}}
```

### DNS-over-TLS

To use DNS-over-TLS without user errors, we must first make our certificate files accessible to the `blocky` user. To do so, copy them to `/opt/blocky`:

```sh
mkdir -p /opt/blocky/certs
cp /etc/letsencrypt/live/{{<hl>}}dns.example.org{{</hl>}}/fullchain.pem /opt/blocky/certs
cp /etc/letsencrypt/live/{{<hl>}}dns.example.org{{</hl>}}/privkey.pem /opt/blocky/certs
```

Ensure the files are accessible by the `blocky` user:

```sh
chown -R blocky:blocky /opt/blocky
```

> To use Blocky as a traditional DNS server in addition to its encrypted functionality, **make sure to run the following command** giving its binary permissions to access port 53:

```sh
setcap CAP_NET_BIND_SERVICE=+eip /opt/blocky/blocky
```

Finally, add your certbot certificates to the `/opt/blocky/config.yaml` file as follows:

```yaml
certFile: /opt/blocky/certs/fullchain.pem
keyFile: /opt/blocky/certs/privkey.pem
``` 

### Blocking Domains

By default, Blocky will **not read** from `/etc/hosts`. Add the following to `config.yaml` to enable this:

```yaml
blocking:
  # Denylists of domains
  denylists:
    hosts:
     - /etc/hosts
  
  # Denylists to enable
  clientGroupsBlock:
    default:
      - hosts
```

If you set a denylist to a URL, blocky will automatically update it for you every 4 hours by default.

```yaml
blocking:
  blackLists:
    hosts:
      - /etc/hosts
    ads:
      - {{<hl>}}https://raw.githubusercontent.com/LukeSmithxyz/etc/master/ips{{</hl>}}
  clientGroupsBlock:
    default:
      - hosts
      - ads
```

### Disabling Logging

Excluding debugging purposes, there is little reason to log user requests on your server. Disable logging by setting `level:` to `error`:

```yaml
log:
  level: error
```

### Starting the Service

After configuring everything to your liking, simply restart the `blocky` service with the following command:

```sh
systemctl restart blocky
```

Congratulations! You've setup a Blocky DNS server!

## Connecting to Blocky on Chrome

All Chromium-based browsers have the option to connect to a DNS-over-HTTPS server.
To enable this, begin by going to [chrome://settings/security](chrome://settings/security), where you will see a "Use secure DNS" option:

![The Google Chrome security settings page.](g1-security.png)

Click on the scroll down menu and select "Add a custom DNS service provider", and insert your DNS server's DNS-over-HTTPS domain in the prompt. This should be `{{<hl>}}https://dns.example.org/dns-query{{</hl>}}`:

![A custom dns in the Google Chrome DNS prompt.](g2-inserting.png)

## Connecting to Blocky on Android

To connect to your Blocky DNS on your Android 9+ phone, begin by opening the settings app and going to the Network & Internet section:

![The settings app, with the "Network & Internet" section highlighted.](1-settings.png)

Then tap on "Private DNS":

![The Network & Internet section with "Private DNS" highlighted.](2-privatedns.png)

In the popup window, type the domain of your DNS server:

![A popup window with "dns.denshi.org" typed into the server option.](3-inserting.png)

If the Private DNS option simply displays your domain name with no errors, you've successfully set up the DNS server!

![Secure DNS server setup with no errors.](4-done.png)

If it instead says "Couldn't Connect", please double-check your configuration and the systemd service to make sure Blocky can access the TLS certificates.

![A couldn't connect error.](5-error.png)
