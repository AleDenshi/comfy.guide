## Guide Information
title: "I2P Daemon"
description: 'Run a website on the invisible internet.'
icon: 'i2p-daemon.svg'
date: 2024-07-07
#ports: [80, 443]

## Author Information
author: "David Uhden"

## Web presense links
links:
  website: "https://github.com/daviduhden"

## Cryptocurrency donation options
crypto:
  xmr: "497pJc7X4xqKvcLBLpSUtRgWqMMyo24u4btCos3cak6gbMkpobgSU6492ztUcUBghyeHpYeczB55s38NpuHoH5WGNSPDRMH"
  btc: "3MDoGJW9TLMTCDGrR9bLgWXfm6sjmgy86f"

draft: true
---

I2P, also known as the "Invisible Internet Project," is a decentralized anonymizing network designed to protect users' privacy and anonymity. I2P uses "garlic routing," a technique that bundles multiple messages together into a single encrypted packet. This offers advantages over onion routing by increasing the efficiency and security of the network, as it is harder for adversaries to correlate packets and trace them back to their source.

## Some Technical Differences Between Tor and I2P

1. **Network Structure**:
   - **Tor**: Uses a centralized directory system to manage and maintain the network. This directory helps users find nodes and establish circuits for their traffic.
   - **I2P**: Employs a fully decentralized network without a central directory. Nodes use a distributed hash table (DHT) to find routes dynamically.

2. **Routing Mechanism**:
   - **Tor**: Utilizes "circuit-based" routing where a predetermined path is set for each connection, and data is passed through this fixed path.
   - **I2P**: Uses "packet-based" routing where each packet may take a different path to reach the destination. This can potentially enhance resilience and reduce latency.

3. **Traffic and Services**:
   - **Tor**: Primarily focuses on enabling anonymous web browsing and offers access to regular websites through its network. It also supports onion services (hidden services) accessible only within the Tor network.
   - **I2P**: Designed to support a wide range of applications including web browsing, email, file sharing, and chat within its network. It provides an internal ecosystem of services, accessible only through I2P.

4. **Protocol Support**:
   - **Tor**: Supports only TCP, which is suitable for web browsing and other TCP-based applications.
   - **I2P**: Supports both TCP and UDP protocols, making it more versatile for different types of applications, including those that require real-time communication or are optimized for UDP.

## Installing I2P

To get the latest version of i2pd, you need to [add the i2pd repositories to your system](https://repo.i2pd.xyz/):

1. Install `apt-transport-https` and `gpg` packages:

    ```sh
    apt install apt-transport-https gpg
    ```

2. Automatically add the repository with a script:

    ```sh
    wget -q -O - https://repo.i2pd.xyz/.help/add_repo | bash -s -
    ```

3. After adding the repository, install i2pd:

    ```sh
    apt update
    apt install i2pd
    ```

## Hidden Services

I2P can proxy any traffic you want to route through it. This includes [webservers](/server/nginx), [Monero nodes](/server/monerod/#i2p-setup), and even [XMPP chat](/server/prosody). This guide will use an NGINX website as an example.

## Configuring I2P

Next, configure the i2pd daemon. The configuration files are located in `/etc/i2pd/`.

1. Edit the `tunnels.conf` file and add the following configuration:

    ```ini
    [example]
    type = http
    host = 127.0.0.1
    port = 8080
    keys = example.dat
    ```

2. You can comment out or remove the default tunnels in the configuration file.

> Tunnels provide a more secure method for using clients over I2P compared to utilizing a SOCKS proxy. Software may still leak sensitive information even when configured to use a SOCKS proxy. This is the reason services hosted over I2P that require a SOCKS proxy are uncommon. For more information, please refer to: [SOCKS and SOCKS proxies](https://geti2p.net/en/docs/api/socks).

### Optional: Generating a Vanity Address

If you run `i2pd` with the above configuration, it will generate a random private key (`example.dat`) for your website at `/var/lib/i2pd/` with a corresponding address made up of 52 random characters.

To pre-generate a private key and create a "vanity" address, install `i2pd-tools`.

1. Clone the `i2pd-tools` repository:

    ```sh
    git clone --recursive https://github.com/purplei2p/i2pd-tools
    ```

2. Install dependencies:

    ```sh
    cd i2pd-tools
    sh dependencies.sh
    ```

3. Compile the tools using `make`:

    ```sh
    make -j$(nproc)
    ```

4. Generate a vanity address using the `vain` tool:

    ```sh
    ./vain example
    ```

    This command will output a new set of private keys named `private.dat`. Copy this file to your i2p configuration:

    ```sh
    cp private.dat /var/lib/i2pd/example.dat
    ```

### Optional: Authentication Strings for Registrars

To register your site with a registrar for a more memorable address, use the `regaddr` tool included in `i2pd-tools`:

1. Generate the authentication string:

    ```sh
    ./regaddr private.dat example.i2p > auth_string.txt
    ```

2. Use the contents of `auth_string.txt` to register your site on a registrar like [http://reg.i2p/add](http://reg.i2p/add) or [http://stats.i2p/i2p/addkey.html](http://stats.i2p/i2p/addkey.html).

## Getting Your I2P Hostname

1. Start and enable the i2pd service:

    ```sh
    systemctl start i2pd
    systemctl enable i2pd
    ```

2. Access your I2P hostname through a command-line browser by visiting `http://127.0.0.1:7070/?page=i2p_tunnels`. You can use a terminal browser such as lynx or w3m.

3. Alternatively, you can use the following command to find your hostname:

    ```sh
    printf "%s.b32.i2p\n" $(head -c 391 /var/lib/i2pd/example.dat | sha256sum | xxd -r -p | base32 | sed s/=//g | tr A-Z a-z)
    ```

    >(If you've generated your own keys for a vanity address, verify that i2pd is reading those keys correctly by checking that the address matches the one generated with the `vain` command.)*

## Adding the NGINX Config

1. Create a new server block configuration file for your site. Open a new file in the `/etc/nginx/sites-available/` directory:

    ```sh
    touch /etc/nginx/sites-available/example
    ```

2. Add the following server block configuration to the file:

    ```nginx
    server {
        listen 127.0.0.1:8080;
        root /var/www/example;
        index index.html;

        location / {
            try_files $uri $uri/ =404;
        }
    }
    ```

3. Enable the site by creating a symbolic link in the `/etc/nginx/sites-enabled/` directory:

    ```sh
    ln -s /etc/nginx/sites-available/example /etc/nginx/sites-enabled/
    ```

5. Restart NGINX to apply the changes:

    ```sh
    systemctl restart nginx
    ```

### Clarifications

Nginx will listen on port 8080, and i2pd will forward your port 8080 to the I2P site port 80. This setup avoids the need to deal with server names.

Now your website should be accessible via the I2P network using your generated I2P hostname.

## Configuring a Firefox-based browser to use I2P

> To use I2P on a Linux desktop OS, it is necessary to install and start/enable the i2pd daemon.

To configure your Firefox-based browser for use with I2P, follow these steps:

1. Click the hamburger menu (three horizontal lines) in the top right corner of your browser.
![](firefox-browser-config-1)
2. Click the small search bar, search for "network settings," and click "Settings" on the right of "Network Settings."
![](firefox-browser-config-2)
3. Enable "Manual proxy configuration."
4. Set the HTTP proxy to `127.0.0.1` and the port to `4444`.
5. Select "Also use this proxy for HTTPS."
6. Add `localhost` and `127.0.0.1` to the "No proxy for" field, then save your changes by clicking "OK."
![](firefox-browser-config-3)
7. Go to "Privacy and Security" in the settings, scroll down to "HTTPS-Only Mode," and select "Don't enable HTTPS-Only Mode." This will prevent annoying "No HTTPS version available" screens when browsing I2P sites.
![](firefox-browser-config-4)
8. Open a new tab and enter `about:config` in the address bar.
9. Search for `media.peerConnection.ice.proxy_only` and set it to `true`.
![](firefox-browser-config-5)
10. Search for `network.dns.disableIPV6` and set it to `true`. This will prevent your ISP from seeing the root URL of the I2P sites you visit.
![](firefox-browser-config-6)
11. Visit [http://notbob.i2p](http://notbob.i2p) to test if everything works correctly.
![](firefox-browser-config-7)