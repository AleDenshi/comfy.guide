---
## Guide Information
title: SQL
date: 2023-01-01
description: A collection of guides on installing and managing SQL databases on the server.
icon: psql.svg

## Author Information
author: Denshi
---

SQL is a standardized language for database management. This article is a "cheat sheet" of sorts, helping you with the semantics of database engines such as MySQL and PostgreSQL.

## Note on Encoding

While a manual server setup will normally ensure a proper locale is installed, some VPS providers and default server ISOs may need further configuration to make sure all data is encoded correctly on your server. Before installing any database engine, ensure your server is using the proper encoding locale.

Simply run the command `dpkg-reconfigure locales` and select the desired locale from the text menu. `en_US.UTF-8` is the default for the english-speaking US.

> Reboot for these changes to take effect.

## MySQL
MySQL is one of the most popular and oldest database software still around today in modern hosting. **MariaDB** is a community-developed fork of MySQL which works almost exactly the same, but with additional security and stability improvements.

### Basic operation
To enter the MySQL prompt, run `mysql`.

### Creating a user

```sql
CREATE USER '{{<hl>}}username{{</hl>}}'@'localhost' IDENTIFIED BY '{{<hl>}}password{{</hl>}}';
```

### Creating a database

```sql
CREATE DATABASE {{<hl>}}database_name{{</hl>}};
```

### Granting permissions

```sql
use {{<hl>}}database_name{{</hl>}};
GRANT ALL ON {{<hl>}}database_name{{</hl>}}.* TO '{{<hl>}}username{{</hl>}}'@'localhost';
```

### Listing existing databases

```sql
SHOW DATABASES;
```

### Deleting a database

```sql
DROP DATABASE {{<hl>}}database_name{{</hl>}};
```

## PostgreSQL
PostgreSQL is a more extensible SQL database, common in use with lots of server software like [PeerTube](/server/peertube) and [Matrix Synapse](/server/matrix).

### Basic operation
The `psql` prompt is only accessible through the `postgres` user, so firstly switch to using it:

```sh
sudo su postgres
```

### Creating a user
User creation in PostgreSQL is done **outside of the `psql` prompt.** The `createuser` command is used, by the `postgres` user:

```sh
createuser --pwprompt {{<hl>}}username{{</hl>}}
```

### Deleting a user

```sh
dropuser {{<hl>}}username{{</hl>}}
```

### Dumping a database

```sh
pg_dump -U {{<hl>}}username{{</hl>}} {{<hl>}}database_name{{</hl>}} > {{<hl>}}file.sql{{</hl>}}
```

#### Restoring from a file

```sh
psql -U {{<hl>}}username{{</hl>}} {{<hl>}}database_name{{</hl>}} < {{<hl>}}file.sql{{</hl>}}
```
Some PostgreSQL operations are performed by the `postgres` user, such as:

Some operations are done from the prompt. Run this command to enter the PostgreSQL prompt:

```sh
psql
```

### Listing existing databases

Simply run `\l` in the `psql` prompt to list existing databases.

```sql
\l
```

### Creating a database

Just like in regular SQL, the `CREATE DATABASE` command is used, but the syntax for ownership is different:

```sql
CREATE DATABASE {{<hl>}}database_name{{</hl>}}
OWNER {{<hl>}}username{{</hl>}};
```

### Deleting a database

Just like regular SQL, run `DROP DATABASE`:

```sql
DROP DATABASE {{<hl>}}database_name{{</hl>}}
