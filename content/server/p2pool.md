---
## Software Information
title: P2Pool
date: 2024-07-07
icon: p2pool.svg
description: Mine Monero on the decentralized P2Pool network.
ports: [18081, 37889, 37888, 3333]

## Author Information
author: Denshi
---

[P2Pool](https://github.com/SChernykh/p2pool) is a decentralized service to mine [Monero.](/server/monerod) It works by making users attach the p2pool software to their existing Monero nodes, and mining through the port provided by the p2pool daemon.

## The need for P2Pool

By default, Monero mining on your own is extremely unlikely to yield results unless you have a significant amount of computing power. By using a mining pool, you can organize with other users and mine together, splitting the rewards. However, this comes with a myriad of restrictions depending on the pool owner, such as *fees, high minimum payouts* and the risk of *centralized 51% attacks* on the wider Monero network.

P2Pool solves this by offering a decentralized network to distribute mining rewards based on contribution. It's a mining pool you connect to with the *p2pool software,* and touts various advantages such as: **0% fees, a minimum payout of ~0.00027 XMR,** and most importantly **pure decentralized mining.**

## Configuring `monerod`

To run P2Pool, you need to already have your own [Monero node.](/server/monerod) P2Pool requires the exposure of a ZMQ interface. Make the following changes to `/etc/monerod.conf`:

```sh
# P2Pool config
zmq-pub=tcp://127.0.0.1:{{<hl>}}18083{{</hl>}}
out-peers=64
in-peers=32
disable-dns-checkpoints=1
```

The port used here is `18083`, but if this port is already being used (for example, by a hidden service) then edit the configuration to a different port.

With `/etc/monerod.conf` edited correctly, simply restart the Monero daemon:

```sh
systemctl restart monerod
```

## Installation

Begin by downloading and extracting the binary:

```sh
baseurl="https://github.com/SChernykh/p2pool/releases"
ver=$(basename "$(curl -w "%{url_effective}\n" -I -L -s -S $baseurl/latest -o /dev/null)")

arch=$(uname -m)
case $arch in
    x86_64)
        arch="x64"
        ;;
    aarch64)
        arch="aarch64"
        ;;
    *)
        exit 1
        ;;
esac

curl -fLO "https://github.com/SChernykh/p2pool/releases/download/$ver/p2pool-$ver-linux-$arch.tar.gz"
tar xvf p2pool*
```

Create a directory for `p2pool` in `/opt/` and place the binary in there:
```sh
mkdir -p /opt/p2pool
mv p2pool*/p2pool /opt/p2pool
cd /opt/p2pool
```


## Configuration

Begin by creating a `p2pool` user and giving it permissions over the directory:

```sh
useradd p2pool -d /opt/p2pool
chown -R p2pool:p2pool /opt/p2pool
```

As the `p2pool` user, begin synchronizing the p2pool side-chain:

```sh
su -c "/opt/p2pool/p2pool --host 127.0.0.1 --wallet {{<hl>}}your_monero_wallet_address{{</hl>}}" p2pool
```

> Please note: The Monero address in the node is what all payments will be made out to. Specifying different addresses in mining software like XMRig will not affect this.

> Initial syncronization may take a little under half an hour.

Now create a systemd service for p2pool in `/etc/systemd/system/p2pool.service`:

```systemd
[Unit]
Description=P2Pool Daemon
After=syslog.target
After=network.target

[Service]
RestartSec=2s
Type=simple
User=p2pool
Group=p2pool
WorkingDirectory=/opt/p2pool
ExecStart=/opt/p2pool/p2pool --loglevel 0 --host 127.0.0.1 --wallet {{<hl>}}your_monero_wallet_address{{</hl>}}
Restart=always

[Install]
WantedBy=multi-user.target
```

To start the service, run the following:
```sh
systemctl enable --now p2pool
```

## Mining

To mine, begin by installing [XMRig](https://xmrig.com/) to your computer.

```sh
./xmrig -o {{<hl>}}your_server_ip{{</hl>}}:3333
```

## Monitoring the Node

Make sure your node is publicly reachable by putting your p2pool node's IP address into [2p2pool observer:](https://p2pool.observer/)

![The P2Pool observer page.](01-check.png)

Congratulations! You've successfully set up a P2Pool node.
