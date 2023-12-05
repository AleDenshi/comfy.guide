---
## Software Information
title: KISS Linux
date: 2023-11-28
description: A minimal source-based Linux distribution.
icon: kisslinux.svg

## Author Information
author: Denshi
---

[KISS Linux](https://kisslinux.org) is a Linux distribution that uses the `kiss` package manager, a simple extensible shell script. KISS Linux is source-based, meaning packages are downloaded and built from source rather than prebuilt.

KISS Linux is **community maintained.** It was originally developed by [Dylan Araps](https://github.com/dylanaraps) who left the project on december 13th, 2021. The repositories and the `kiss` package manager have since been maintained by [kiss-community](https://kisscommunity.org/), which hosts the repositories on [Codeberg.](https://codeberg.org/kiss-community)

## Before you begin...

**Source-based distributions can be installed from any other distribution.** Flash a separate (optionally graphical) Linux image on a CD/DVD/USB and follow the commands below to install KISS. Some good images to use are [Linux Mint](linuxmint.com) and [Artix Linux.](https://artixlinux.org)

## Choosing between KISS and GKISS

*Throughout this tutorial, different repository links will be provided depending on whether you wish to install KISS or GKISS Linux.*

KISS is maintained in two different forms: Regular KISS Linux and GKISS Linux. In short, GKISS is the more "compatible" of the two because of its usage of the [GNU C Library (glibc)](https://www.gnu.org/software/libc/). For example, [NVIDIA drivers](/client/nvidia) function on GKISS, while they do not on KISS. Regular KISS Linux uses [musl libc](https://musl.libc.org/) instead, which may not be compatible with as much software.

## Partitioning and Mounting

Before installing the system, run `cfdisk` to properly partition the space you wish to use for KISS. This guide will assume you're installing to `/dev/sda1`.

Format `/dev/sda1` to the EXT4 filesystem:

```sh
mkfs.ext4 /dev/sda1
```

Mount `/dev/sda1` to `/mnt`:

```sh
mount /dev/sda1 /mnt
```

## Downloading the Tarball

Begin by downloading the latest KISS tarball:

```sh
# Repository to use. Pick "" for KISS, "g" for GKISS.
prefix=""

# Get version information
baseurl="https://codeberg.org/kiss-community/${prefix}repo/releases"
ver=$(basename "$(curl -w "%{url_effective}\n" -I -L -s -S $baseurl/latest -o /dev/null)")
file=${prefix}kiss-chroot-$ver.tar.xz
url="$baseurl/download/$ver/$file"

# Download the tarball
curl -fLO "$url"
```

## Extracting the Tarball and Chrooting

Extract the tarball to `/mnt`:

```sh
tar xvf $file -C /mnt
```

KISS Linux tarballs come with a built-in chroot script named `kiss-chroot`. Simply run this and you'll immediately be in the new filesystem:

```sh
/mnt/bin/kiss-chroot /mnt
```

## Repository Setup

KISS repositories should be stored in `/var/db/kiss/`. Clone the relevant repositories to this directory:

```sh
# Repositories for KISS Linux
git clone https://codeberg.org/kiss-community/repo /var/db/kiss/repo

# Repositories for GKISS LINUX
git clone https://codeberg.org/kiss-community/grepo /var/db/kiss/repo

# Community repository
git clone https://codeberg.org/kiss-community/community /var/db/kiss/community
```

The repositories contain various subdirectories (`core`, `extra` and `wayland`) which themselves contain the package sources and `kiss` package configurations.

The KISS package manager only uses the repositories it finds in the `KISS_PATH` variable. To declare this variable on the system level, add it to `/etc/profile`:

```sh
export KISS_PATH=''
KISS_PATH=/var/db/kiss/repo/core
KISS_PATH=$KISS_PATH:/var/db/kiss/repo/extra
KISS_PATH=$KISS_PATH:/var/db/kiss/repo/wayland

# Community repository
KISS_PATH=$KISS_PATH:/var/db/kiss/community/community
```

For the changes to take effect, run:

```sh
source /etc/profile

# Check the KISS_PATH variable
echo $KISS_PATH
```

## Package Building Configuration

In addition to declaring variables for the package manager, `/etc/profile` should also contain the following to tune package compilation:

```sh
export CFLAGS="-O3 -pipe -march=native"
export CXXFLAGS="$CFLAGS"

# NOTE: '4' should be changed to match the number of threads.
# 	This value can be found by running 'nproc'.
export MAKEFLAGS="-j4"
```

## Updating and Rebuilding

Now you can update the repositories and rebuild all installed packages:
```sh
kiss update
cd /var/db/kiss/installed && kiss build *
```
