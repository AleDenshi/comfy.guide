---
## Software Information
title: P2Pool
date: 2023-09-16
draft: true
icon: p2pool.svg
description: Mine Monero on the decentralized P2Pool network.
ports: [18081, 37889, 37888]

## Author Information
author: Denshi
---

[P2Pool](https://github.com/SChernykh/p2pool) is a decentralized way to mine [Monero.](/server/monerod)

## The need for P2Pool

By default, Monero mining on your own is extremely unlikely to yield results unless you have a significant amount of computing power. By using a mining pool, you can organize with other users and mine together, splitting the rewards. However, this comes with a myriad of restrictions depending on the pool owner, such as *fees, high minimum payouts* and the risk of *centralized 51% attacks* on the wider Monero network.

P2Pool solves this by offerring a decentralized network to distribute mining rewards based on contribution. It's a mining pool you connect to with the *p2pool software,* and touts various advantages such as: **0% fees, a minimum payout of ~0.00027 XMR,** and most importantly **pure decentralized mining.**

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

Begin by installing the build requirements for p2pool:

```sh
apt update
apt install git build-essential cmake libuv1-dev libzmq3-dev libsodium-dev libpgm-dev libnorm-dev libgss-dev libcurl4-openssl-dev libidn2-0-dev
```

Then clone the repository and build p2pool:

```sh
git clone --recursive https://github.com/SChernykh/p2pool
cd p2pool
mkdir build && cd build
cmake ..
make -j$(nproc)
```

Create a directory for `p2pool` in `/opt/` and place the binary in there:
```sh
mkdir -p /opt/p2pool
cp p2pool /opt/p2pool
cd /opt/p2pool
```

## Configuration
