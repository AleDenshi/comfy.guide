---
## Guide Information
title: Prosody
date: 2022-04-03
icon: prosody.svg
description: A minimalist XMPP chat server.
ports: [5000, 5222, 5269, 5280, 5281]

## Author Information
author: Denshi
---

XMPP is a fantastically simple protocol that's usually used as a messenger. It's highly extensible, better than IRC, lighter and more decentralized than Matrix, and normie social media like Telegram can't hold a candle to it.

XMPP is so decentralized and extensible that there are many [*different*](/server/ejabberd) XMPP servers. Here, let's set up a [Prosody](https://prosody.im/) XMPP server.

## Installation

To install Prosody, first add the official Prosody repositories for Debian:

```sh
# Install extrepo if you already haven't
apt install extrepo
extrepo enable prosody
apt update
```
Then, install Prosody:

```sh
apt install prosody
```

## Configuration

The Prosody configuration file is in `/etc/prosody/prosody.cfg.lua`. To set it all up, we will be changing several things.

### Setting Admins

Let's go ahead and set who our admin(s) will be. Find the line that says `admins = { }` and to this we can specify one or more server admins.

```lua
-- To add one admin:
admins = { "admin@example.org" }

-- We can add more than one by separating them by commas. (This file is written in Lua.)
admins = { "admin1@example.org", "admin2@example.org" }
```

Note that we have not created these accounts yet, we will do this [below](#user).

### Setting the VirtualHost

Find the line `VirtualHost "localhost"` and replace `localhost` with your domain. In our case, we will have `VirtualHost "example.org"`

### Database Setup

Prosody includes the `internal` and `sql` storage back-ends by default. 
If you wish to run Prosody with PostgreSQL, begin by installing the PostgreSQL and the corresponding Lua library:

```sh
apt install postgresql lua-dbi-postgresql
```

Then start the daemon:

```sh
systemctl restart postgresql
```

Now create a user named `prosody` to manage your database:

```sh
su -c "createuser --pwprompt prosody" postgres
```

And finally, create the actual database:

```sh
su -c "psql -c 'CREATE DATABASE prosody OWNER prosody;'" postgres
```

Finally, in `/etc/prosody/prosody.cfg.lua`, edit the following lines:

```lua
storage = "sql"

sql = {
    driver = "PostgreSQL",
    database = "{{<hl>}}prosody{{</hl>}}",
    username = "{{<hl>}}prosody{{</hl>}}",
    password = "{{<hl>}}password{{</hl>}}",
    host = "localhost"
}
```

### Message Archive Management (Chat History)

By default, Prosody will send out messages received only to the first available clients.
That means that if you have your desktop client turned off and your cell phone receives a message,
it will *not* be available to the desktop client when you start it.

While this may be preferred in some cases,
enable the `mam` module (Message Archive Management) to have the server hold on messages and sync them to all clients.

Within the `modules_enabled` block, you can uncomment the `mam` line to enable it:

```lua
modules_enabled = {
...
            "mam"; -- Store recent messages to allow multi-device synchronization
...
}
```

You can see other settings for this module [here.](https://prosody.im/doc/modules/mod_mam)
Like, for example, how long a server should hold on to message histories for syncing.

Note also that Prosody comes with the `carbons` activated module by default, which is related. This will send received messages to *all* active clients (your phone and desktop), although it will not save messages like MAM for clients not online or to be added later.

### Voice and Video Calling

Prosody supports XMPP voice and video calls through an external TURN and STUN server.

First, follow the guide on installing and setting up [coturn,](/server/coturn) setting  **only a shared secret** for authentication.

Then, uncomment the `turn_external` module in the modules section in `prosody.cfg.lua`.

```lua
"turn_external";
```

Finally, specify the host and credentials lower in the config:

```lua
-- Specify the address of the TURN service (you may use the same domain as XMPP)
turn_external_host = "{{<hl>}}turn.example.org{{</hl>}}"

-- This secret must be set to the same value in both Prosody and the TURN server
turn_external_secret = "{{<hl>}}your shared secret{{</hl>}}"
```
### Enable Registration

If you want to run a general public XMPP server, you can allow anyone to create an account by changing the following in `/etc/prosody/prosody.cfg.lua`:

```lua
allow_registration = true
```

### Abuse/Admin Contact Info

Prosody allows you to set and publish a list of contacts for any abuse or admin complaints your server may get. **This is not essential for a functioning server,** but you may want to set this if you plan on running a public server.

```lua
contact_info = {
    -- You can specify email addresses as well as XMPP addresses.
    abuse = { "xmpp:admin@example.org", "mailto:admin@example.org" };
    admin = { "xmpp:admin@example.org", "mailto:admin@example.org" };
}
```

## Components

Prosody makes use of various "components", all hosted through separate domains/subdomains of your main server, to host a **variety of useful features** to make your XMPP experience as complete as possible.

### Multi-User Chats

Most people will probably want the ability to have chats with more than two users. This is easily enough to enable. In the config file, add the following:

```lua
Component "{{<hl>}}chat.example.org{{</hl>}}" "muc"

    -- This muc_mam module keeps backups of multi-user chats
    modules_enabled = { "muc_mam" }
    
    -- Restrict room creation to a specific set of users
    restrict_room_creation = "admin"
```

On the first line, you must have a separate subdomain for your multi-user chats. The `chat.` subdomain is used here, but some servers use `muc.`. Anything is possible.

> The `restrict_room_creation` line important because it prevents non-admins from creating and squatting rooms on your server. The only situation where you might not want that is if you intend to open a general public chat system for people you don't know.

Read more about the `muc` plugin on the Prosody documentation page [here](https://prosody.im/doc/modules/mod_muc).

### File sharing

With this we can bring XMPP to the level of other popular instant messaging applications like Matrix and WhatsApp.
It is extremely easy to setup.
This part is optional, but it can make XMPP more normie-friendly if you plan on moving family members and friends over to XMPP.

Add the following line to you prosody config file to enable file uploads:

```lua
Component "{{<hl>}}upload.example.org{{</hl>}}" "http_file_share"
```

A big concern with file sharing is large files, seeing as all files shared over XMPP will be stored on your server. This can become a problem when many (and large) files are being shared. We can put a cap on large files by adding the following line to our config:

```lua
http_file_share_size_limit = 20971520
```

This puts a 20MB cap on all files being shared. The value is specified in bytes. You can also specify after how long files should be deleted by adding the following line:

```lua
http_file_share_expire_after = 60 * 60 * 24 * 7
```

The value is specified in seconds. The above line will make prosody delete files after a week.

### Proxy Support

This helps with file transfers for devices behind a NAT, and unless you are using XMPP in a LAN, you **probably need this.**
Enable the proxy by adding the following line to the config:

```lua
Component "{{<hl>}}proxy.example.org{{</hl>}}" "proxy65"
```

At this point, file sharing is now setup and ready for pretty much all use cases.

### Pub-Sub Support {#pubsub}

Publish-Subscribe (also known as Pubsub) is a simple protocol that allows data to be published to "nodes"; Think of these as mini-blogs that can be published over XMPP. New posts on these nodes will be automatically pushed to subscribers directly, unlike RSS which requires the user to poll the feed itself.

To setup a Pubsub component, add the following lines to your Prosody config:

```lua
Component "{{<hl>}}pubsub.example.org{{</hl>}}" "pubsub"
```

Having a Pubsub component will allow your server to participate in the wider XMPP Pubsub social network, mainly utilized by the [Movim](https://movim.eu) client.

## Certificates

Obviously, we want to have client-to-server and server-to-server encryption. We can use Certbot to generate certificates and use a convenient `prosodyctl` command to import them.

For added security, begin by installing the `lua-unbound` library for DNS queries:
```sh
apt install lua-unbound
```

Run the following certbot command. Include the `--nginx` if you also have an [NGINX server](/server/nginx) running.

```sh
certbot -d {{<hl>}}example.org{{</hl>}} --nginx
```

It's essential to obtain **any other certificates** for [additional services](#components) on your server. For example, here are two certificates for subdomains hosting multi-user chats, file sharing, proxy and a Pub-Sub service respectively:

```sh
certbot -d {{<hl>}}chat.example.org{{</hl>}} --nginx
certbot -d {{<hl>}}uploads.example.org{{</hl>}} --nginx
certbot -d {{<hl>}}proxy.example.org{{</hl>}} --nginx
certbot -d {{<hl>}}pubsub.example.org{{</hl>}} --nginx
```

Once you have all your certificates for encryption, run the following to import them into Prosody:

```sh
prosodyctl --root cert import /etc/letsencrypt/live/
```

> You can re-run the `prosodyctl --root cert import` command again when you need to renew or change certificates. It will always attempt to get the latest certificates needed by your current configuration in `/etc/prosody/prosody.cfg.lua`, and will inform you if it can't access them.

Note that you might get an error that a certificate has not been found if your `muc` subdomain and your main domain share a certificate. It should still work, this is just notifying you that no specific certificate for the subdomain.

**Note:** The above command will need to be rerun when certificates are renewed. You may want to create a [cronjob](/server/cron) to have this done automatically.

## Included Modules

Prosody includes a variety of modules out of the box to enable and configure.

### Bosh and Websockets

These two modules allow web-based clients to interface with your server. Begin by installing the required bitop library:

```sh
apt install lua-bitop
```

Then enable the module under `modules_enabled`:
```lua
modules_enabled = {
...
        "bosh"; -- Enable BOSH clients, aka "Jabber over HTTP"
        "websocket"; -- XMPP over WebSockets
...
}
```

### Tombstones

This module leaves a "tombstone" of a users account after it is deleted. This makes it impossible for another user to register an account and impersonate the original user and the original account was deleted.

```lua
modules_enabled = {
...
        "tombstones"; -- Prevent registration of deleted accounts
...
}
```

### Mimicking

Because unicode is prone to similarly-looking characters that can be used to spoof usernames, this module helps equate these and prevent people from making similar usernames. Simply enable it in `/etc/prosody/prosody.cfg.lua`:

```lua
modules_enabled = {
...
        "mimicking"; -- Prevent address spoofing
...
}
```

To send announcements

## Community Modules

Prosody configuration can go far beyond the included modules. There are many [community modules](https://modules.prosody.im/), and Prosody comes with a community module installer built-in on version 0.12 and above.

To install community modules, you need to first install [Luarocks](https://luarocks.org/):

```sh
apt install luarocks
```

### Push Notifications

Push notifications are especially useful for iOS devices that require push support to continually receive notifications. Otherwise iOS users would need to keep their XMPP client constantly in the foreground.

Begin by enabling the `smacks` module, to ensure full compatibility:

```lua
modules_enabled = {
...
            "smacks"; -- Stream management and resumption (XEP-0198)
...
}
```

Then install the `mod_cloud_notify` and `mod_cloud_notify_extensions` modules:

```sh
prosodyctl install --server=https://modules.prosody.im/rocks/ mod_cloud_notify
prosodyctl install --server=https://modules.prosody.im/rocks/ mod_cloud_notify_extensions
```

The `cloud_notify_extensions` module requires the `luaossl` package to support encrypted push notifications, so make sure to install this package on Debian:

```sh
apt install lua-luaossl
```

Now simply enable the modules in `modules_enabled` like any other module:

```lua
modules_enabled = {
...
            "cloud_notify";
            "cloud_notify_extensions";
...
}
```

### MUC Avatar/Vcard Support

If you want users to be able to publish avatars in group chats (without having to add each other as contacts), then install this community module and enable it under your `muc` component:

```sh
prosodyctl install --server=https://modules.prosody.im/rocks/ mod_vcard_muc
```

```lua
Component "{{<hl>}}chat.example.org{{</hl>}}" "muc"
...
    modules_enabled = { "muc_mam", "vcard_muc" }
...
```

### Pubsub RSS Feed Mirroring

If you've setup a [Pubsub component](#pubsub), you can enable a community module to automatically grab RSS feeds and publish their contents to a Pubsub node of your choosing. Simply install the community module:

```sh
prosodyctl install --server=https://modules.prosody.im/rocks/ mod_pubsub_feeds
```

And enable it under the Pubsub component:

```lua
Component "{{<hl>}}pubsub.example.org{{</hl>}}" "pubsub"

-- Enable the module here
modules_enabled = { "pubsub_feeds" }

feeds = {
  -- The part before = is used as PubSub node
  denshiblog = "https://denshi.org/index.xml";
  lukesmith = "https://lukesmith.xyz/index.xml";
  planet_jabber = "https://planet.jabber.org/atom.xml";
  prosody_blog = "https://blog.prosody.im/feed/atom.xml";
}

-- You can also set a custom time interval for when to grab new RSS content
-- (default is 900 seconds, 15 minutes)
feed_pull_interval_seconds = 900
```

#### Granting Ownership on Module-Created Pubsub Nodes

By default, Pubsub nodes created by the `pubsub_feeds` module will not be owned by anyone, not even the server admin. To change this, use the Prosody shell to run a lua command using [`util.pubsub`](https://prosody.im/doc/developers/util/pubsub) granting you ownership:

```sh
## Enter the Prosody shell
prosodyctl shell
```

```lua
-- Run the command with the > at the beginning
>prosody.hosts["{{<hl>}}pubsub.example.org{{</hl>}}"].modules.pubsub.service:set_affiliation("{{<hl>}}node_name{{</hl>}}",true,"{{<hl>}}user@example.org{{</hl>}}","owner")

```

## Creating users/admins manually {#user}

Let's manually create the admin user we prepared for above. Note that you can indeed do this in your XMPP client if you have not disabled registration, but this is how it is done on the command line:

```sh
prosodyctl adduser {{<hl>}}admin@example.org{{</hl>}}
```

This will prompt you to create a password as well.

## Make changes active

With any system service, use `systemctl reload` or `systemctl restart` to make the new settings active:

```sh
systemctl restart prosody
```

## Using your Server

Once your server is set up, you just need an **XMPP client** to use your new and secure chat system.
