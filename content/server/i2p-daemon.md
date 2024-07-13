## Guide Information
title: "I2P Daemon"
description: 'Run a website on the invisible internet.'
icon: 'i2p-daemon.svg'
date: 2024-07-07
ports: [4440, 8080]

## Author Information
author: "David Uhden"

## Web presense links
links:
  website: "https://github.com/daviduhden"

## Cryptocurrency donation options
crypto:
  xmr: "497pJc7X4xqKvcLBLpSUtRgWqMMyo24u4btCos3cak6gbMkpobgSU6492ztUcUBghyeHpYeczB55s38NpuHoH5WGNSPDRMH"
  btc: "3MDoGJW9TLMTCDGrR9bLgWXfm6sjmgy86f"

---

I2P, also known as the "Invisible Internet Project," is a decentralized anonymizing network designed to protect users' privacy and anonymity. I2P uses "[garlic routing](https://geti2p.net/en/docs/how/garlic-routing)", a technique that bundles multiple messages together into a single encrypted packet. This offers advantages over onion routing by increasing the efficiency and security of the network, as it is harder for adversaries to correlate packets and trace them back to their source.

## Some Technical Differences Between Tor and I2P

1. **Network Structure**:
   - **Tor**: Uses a centralized directory system to manage and maintain the network. This directory helps users find nodes and establish circuits for their traffic.
   - **I2P**: Employs a fully decentralized network without a central directory. Nodes use a distributed hash table (DHT) to find routes dynamically.

2. **Network Participation**:
   - **Tor**: Users are only clients by default, which means that they do not relay traffic unless they configure their node to act as a relay or exit node, which requires manual configuration.
   - **I2P**: All users act as nodes by default, which means that they automatically share bandwidth to help route traffic for other users, enhancing the overall capacity and resilience of the network through decentralization.

3. **Routing Mechanism**:
   - **Tor**: Utilizes "circuit-based" routing where a predetermined path is set for each connection, and data is passed through this fixed path.
   - **I2P**: Uses "packet-based" routing where each packet may take a different path to reach the destination. This can potentially enhance resilience and reduce latency.

4. **Traffic and Services**:
   - **Tor**: Primarily focuses on enabling anonymous web browsing and offers access to regular websites through its network. It also supports onion services (hidden services) accessible only within the Tor network.
   - **I2P**: Designed to support a wide range of applications including web browsing, email, file sharing, and chat within its network. It provides an internal ecosystem of services, accessible only through I2P.

5. **Protocol Support**:
   - **Tor**: Supports only TCP, which is suitable for web browsing and other TCP-based applications.
   - **I2P**: Supports both TCP and UDP protocols, making it more versatile for different types of applications, including those that require real-time communication or are optimized for UDP.

## Installing I2Pd

To get the latest version of i2pd, you need to [add the i2pd repositories to your system](https://repo.i2pd.xyz/).

1. Install `apt-transport-https` and `gpg` packages:

	```sh
	apt install apt-transport-https gpg
	```

2. Automatically add the repository with a bash script:

	```sh
	wget -q -O - https://repo.i2pd.xyz/.help/add_repo | bash -s -
	```

3. After adding the repository, install i2pd:

	```sh
	apt update
	apt install i2pd
	```

4. Start and enable the i2pd service:

	```sh
	systemctl start i2pd
	systemctl enable i2pd
	```

## Hidden Services

I2P can proxy any traffic you want to route through it. This includes [webservers](/server/nginx) (also called eepsites), [Monero nodes](/server/monerod/#i2p), and even [XMPP chat](https://i2pd.readthedocs.io/en/latest/tutorials/xmpp/). This guide will use an NGINX website as an example.

## Configuring I2Pd

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

> Tunnels provide a more secure method for using clients over I2P compared to utilizing a SOCKS proxy. Software may still leak sensitive information even when configured to use a SOCKS proxy. This is the reason services hosted over I2P that require a SOCKS proxy are uncommon. For more information refer to the [SOCKS and SOCKS proxies](https://geti2p.net/en/docs/api/socks) article.

### Improving connectivity

If your system has a firewall or you are behind a NAT you should configure a port for incoming connections.

1. Edit the `i2pd.conf` file located in `/etc/i2pd/` and look for this section:

	```ini
	## Port to listen for connections
	## ...
	# port = 4567
	```

2. Uncomment the `port` option and change its value to a [port number that is not used](https://en.wikipedia.org/wiki/List_of_TCP_and_UDP_port_numbers) by any service (such as `4440`).

	```ini
	## Port to listen for connections
	## ...
	port = 4440
	```
3. Allow TCP and UDP connections on the selected port in your firewall:

	```sh
	ufw allow in 4440/tcp
	ufw allow in 4440/udp
	```
	
4. If you are behind a NAT you should configure your router rules accordingly.

### Optional: Generating a Vanity Address

If you run `i2pd` with the above configuration, it will generate a random private key (`example.dat`) for your website at `/var/lib/i2pd/` with a corresponding address made up of 52 random characters.

To pre-generate a private key and create a "vanity" address, install [i2pd-tools](https://github.com/purplei2p/i2pd-tools).

1. Clone the `i2pd-tools` repository:

	```sh
	git clone --recursive https://github.com/purplei2p/i2pd-tools
	cd i2pd-tools
	```

2. Install dependencies:

	```sh
	sh dependencies.sh
	```

3. Compile the tools using `make`:

	```sh
	make
	```

4. Generate a vanity address using the `vain` tool:

	```sh
	./vain web
	```
> Please note: The more characters you add to your filter, the longer the program will take to generate the address! Running it with `web` may only take a few seconds, but running it with `website` may take weeks!

This command will output a new set of private keys named `private.dat`. Copy this file to `/var/lib/i2pd/`:

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

> You can also register subdomains with the `regaddr_3ld` tool to host other services under the same memorable address. For more information refer to the `i2pd-tools` [documentation on regaddr_3ld](https://github.com/purplei2p/i2pd-tools?tab=readme-ov-file#regaddr_3ld).

## Getting Your I2P Hostname

1. Restart the i2pd service:

	```sh
	systemctl restart i2pd
	```

2. Access your I2P hostname through a command-line browser by visiting `http://127.0.0.1:7070/?page=i2p_tunnels`. You can use a terminal browser such as lynx or w3m.

3. Alternatively, you can use the following command to find your hostname:

	```sh
	printf "%s.b32.i2p\n" $(head -c 391 /var/lib/i2pd/example.dat | sha256sum | xxd -r -p | base32 | sed s/=//g | tr A-Z a-z)
	```

>If you've generated your own keys for a vanity address, verify that i2pd is reading those keys correctly by checking that the address matches the one generated with the `vain` command.

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

4. Restart NGINX to apply the changes:

	```sh
	systemctl restart nginx
	```

### Clarifications

I2Pd will listen for HTTP requests on port 4440 and forward them internally to NGINX on port 8080 through the tunnel configured previously. This configuration avoids the need to deal with server names.

Now your website should be accessible via the I2P network using your generated base32 I2P hostname, or at the address registered with the registrar, but note that the latter may take some time to be accessible as the addressbook has to be updated in the clients.

## Connecting to your website

The I2Pd browser is a pre-configured version of Firefox ESR for its use on the I2P network, it is the fastest and easiest way to start surfing the web on the Invisible Internet.

> To use this browser you need to have previously [installed](https://i2pd.readthedocs.io/en/latest/user-guide/install/#linux) I2Pd on your Linux system. 

To use this browser you have to follow these simple steps.

1. Install the dependencies if they are not already installed.

	```sh
	apt install curl tar screen
	```
	
3. Clone the `i2pd-browser` repository:

	```sh
	git clone https://github.com/daviduhden/i2pd-browser/
	cd i2pd-browser
	```

4. Build the pre-configured Firefox using the `build` shell script from the `build` directory:

	```sh
	cd build
	./build
	```

5. Run I2Pd by executing the `i2pd` shell script from `i2pd` directory:

	```sh
	cd ../i2pd
	./i2pd
	```

6. Run Firefox by executing the `start-i2pd-browser.desktop` desktop entry:

	```sh
	cd ../
	./start-i2pd-browser.desktop
	```

7. Now you can access your web page and others on the Invisible Internet, simply enter the base32 address of the web page you have created to verify that it works. I recommend visiting the [notbob directory](http://notbob.i2p/) to find services in the I2P network.

> Unfortunately, the pre-compiled i2pd binaries available for Unix-like operating systems such as Linux and *BSD do not have built-in support for UPnP by default, which makes it very inconvenient for client use. If you want to enable it you need to compile i2pd yourself. For more information refer to the [official documentation](https://i2pd.readthedocs.io/en/latest/devs/building/unix/).
