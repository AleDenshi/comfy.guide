---
## Software Information
title: MongoDB
date: 2023-09-23
description: A simple guide to installing and using the Mongo Database.
icon: mongodb.svg

## Author Information
author: Denshi
---

[MongoDB](https://www.mongodb.com/) is a modern, document-based database that is used by a lot of web software and services. Unlike SQL databases such as [MySQL](/server/sql) that store data in tables, MongoDB utilizes json documents instead.

## Initial Setup
This section covers the base installation of a MongoDB database server; It is highly recommended to follow the steps in further setup.

### Installation
To install a relatively modern version of MongoDB on Debian, it is recommended to use the [MongoDB community edition repositories:](https://docs.mongodb.com/manual/tutorial/install-mongodb-on-debian/)

```sh
curl -fsSL https://pgp.mongodb.com/server-7.0.asc | \
    sudo gpg -o /usr/share/keyrings/mongodb-server-7.0.gpg \
    --dearmor
```

Then update your repository database:

```sh
sudo apt update
```

And install MongoDB:

```sh
sudo apt install mongodb-org
```

### Systemd Service
The `mongodb-org` package automatically installs a [systemd](/server/systemd) service that you can run to start, stop or enable MongoDB at launch:

```sh
sudo systemctl restart mongodb
```

Enabling MongoDB at launch:

```sh
sudo systemctl enable mongodb
```

### Configuration
Configuration of the MongoDB daemon can be done in `/etc/mongod.conf`;
Otherwise, a specific configuration file can be specified with the `--config` option while using `mongod`

> It is recommended to run `mongod` as the user `mongodb`, which is created when installing MongoDB.

## Further Configuration
There are some further recommended steps to take to ensure secure functioning of the MongoDB database:

### Creating a database
To create a database, first access the MongoDB shell:

```sh
mongo
```

Then use the `use` command to create (if not already existing) and begin using a database:

```sh
use {{<hl>}}database-name{{</hl>}}
```

### Enabling Authentication/Access Control
MongoDB supports authentication, but it is disabled by default; To enable it, first enter the MongoDB shell and create a user named `MyUserAdmin` under the `admin` database:

```sh
mongo
use admin
db.createUser(
  {
    user: "myUserAdmin",
    pwd: passwordPrompt(),
    roles: [ { role: "userAdminAnyDatabase", db: "admin" }, "readWriteAnyDatabase" ]
  }
)
```

This will prompt for a password for the admin user; **Remember that password!**

Now exit the MongoDB shell and edit the daemon configuration file at `/etc/mongod.conf` and add the following line under the `security` section:

```yaml
security:
    authorization: enabled
```

Restart the daemon with `sudo systemctl restart mongod` and test authentication:

```sh
mongo --authenticationDatabase "admin" -u "myUserAdmin" -p
```

This will prompt for your password; Enter it, and if you are able to enter MongoDB, you've enabled authentication!

#### Creating a new user
To create a new user, first specify their authentication database:

```sh
use {{<hl>}}database_name{{</hl>}}
```

*Note: [some software](https://gitgud.io/fatchan/jschan) may specifically require you to use the `admin` database to authenticate.)*

Then create the user:

```sh
db.createUser(
  {
    user: "{{<hl>}}username{{</hl>}}",
    pwd:  passwordPrompt(),
    roles: [ { role: "readWrite", db: "{{<hl>}}database-name{{</hl>}}" },
             { role: "read", db: "reporting" } ]
  }
)
```

With this, a user is created.
