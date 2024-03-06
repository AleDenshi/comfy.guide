---
## Software Information
title: KISS Linux
date: 2023-11-28
description: A minimal source-based Linux distribution.
icon: kisslinux.svg

## Author Information
author: Denshi
---

[KISS Linux](https://web.archive.org/web/20240224161022/https://kisslinux.org/) is a Linux distribution that uses the `kiss` package manager, a simple extensible shell script. KISS Linux is source-based, meaning packages are downloaded and built from source rather than prebuilt.

KISS Linux is **community maintained.** It was originally developed by [Dylan Araps](https://github.com/dylanaraps) who left the project on December 13th, 2021. The repositories and the `kiss` package manager have since been maintained by [kiss-community](https://kisscommunity.org/), which hosts the repositories on [Codeberg.](https://codeberg.org/kiss-community)

## Before you begin...

**Source-based distributions can be installed from any other distribution.** Flash a separate (optionally graphical) Linux image on a CD/DVD/USB and follow the commands below to install KISS. Some good images to use are [Linux Mint](linuxmint.com) and [Artix Linux.](https://artixlinux.org)

## Choosing between KISS and GKISS

*Throughout this tutorial, different repository links will be provided depending on whether you wish to install KISS or GKISS Linux.*

KISS is maintained in two different forms: Regular KISS Linux and GKISS Linux. In short, GKISS is the more "compatible" of the two because of its usage of the [GNU C Library (glibc)](https://www.gnu.org/software/libc/). For example, [NVIDIA drivers](/client/nvidia) function on GKISS, while they do not on KISS. Regular KISS Linux uses [musl libc](https://musl.libc.org/) instead, which may not be compatible with as much software.

## Partitioning and Mounting

Before installing the system, run `cfdisk` to properly partition the space you wish to use for KISS. This guide will assume you're installing to `{{<hl>}}/dev/sda1{{</hl>}}`.

Format your root partition to the EXT4 filesystem:

```sh
mkfs.ext4 {{<hl>}}/dev/sda1{{</hl>}}
```

Mount the root partition to `/mnt`:

```sh
mount {{<hl>}}/dev/sda1{{</hl>}} /mnt
```

> Make sure to mount any additional partitions at this point as well. For example, if using UEFI with GPT, mount your FAT 32 boot partition to `/mnt/boot/efi`.

### Downloading the Tarball

Begin by downloading the latest KISS tarball:

```sh
# Repository to use. Pick "" for KISS, "g" for GKISS.
prefix={{<hl>}}""{{</hl>}}

# Get version information
baseurl="https://codeberg.org/kiss-community/${prefix}repo/releases"
ver=$(basename "$(curl -w "%{url_effective}\n" -I -L -s -S $baseurl/latest -o /dev/null)")
file=${prefix}kiss-chroot-$ver.tar.xz
url="$baseurl/download/$ver/$file"

# Download the tarball
curl -fLO "$url"
```

### Extracting the Tarball

Extract the tarball to `/mnt`:

```sh
tar xvf $file -C /mnt
```

### Generating the fstab

The `genfstab` script may be used to generate an `/etc/fstab` file for the KISS Linux install. Download and run the script as follows:

```sh
# Standalone install for all OS':
git clone https://github.com/glacion/genfstab
cd genfstab

./genfstab > /mnt/etc/fstab

```

> Note: Make sure to check the `/mnt/etc/fstab` before rebooting your system at the end of the installation. The script is not foolproof and may have included some unwanted filesystems.

### Changing root

KISS Linux tarballs come with a built-in chroot script named `kiss-chroot`. Simply run this and you'll immediately be in the new filesystem:

```sh
/mnt/bin/kiss-chroot /mnt
```

## Package Configuration

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

### Build Configuration

In addition to declaring variables for the package manager, `/etc/profile` should also contain the following to tune package compilation:

```sh
export CFLAGS="-O3 -pipe -march=native"
export CXXFLAGS="$CFLAGS"

# NOTE: '4' should be changed to match the number of threads.
# 	This value can be found by running 'nproc'.
export MAKEFLAGS="-j4"
```

As with all changes to `/etc/profile`, run the following to put the changes into effect:

```sh
source /etc/profile
```

### Updating and rebuilding

Now you can update the repositories and rebuild all installed packages:
```sh
kiss update
cd /var/db/kiss/installed && kiss build *
```
> If you encounter any issues building packages, make sure your repositories are up to date.
> If the issues persist, do not hesitate to join the kiss-community IRC chat at `#kisslinux` on [libera.chat.](irc://irc.libera.chat)


### Installing necessary packages

Some packages to consider:

- `libelf` -- To build the Linux kernel.
- `perl` -- (Optionally) to build the Linux kernel.
- `baseinit` -- The init system for KISS.
- `grub` -- The recommended bootloader.
- `efibootmgr` -- To boot on UEFI systems.
- `e2fsprogs` -- For EXT4 filesystem support.
- `dosfstools`-- For FAT filesystem support.
- `dhcpcd` -- Needed to connect to a network (DHCP resolution tool).
- `wpa_supplicant` -- For WiFi support.
- `eiwd` -- For slightly easier WiFi support.

To install any of the following commands, run `kiss b` and then `kiss i`:

```sh
kiss b libelf perl baseinit grub e2fsprogs dhcpcd

kiss i libelf perl baseinit grub e2fsprogs dhcpcd
```

## Kernel Configuration

KISS Linux does not come with any preconfigured kernel options. To install and use the Linux kernel, you must download, compile and install it yourself.

Begin by creating the `/usr/src/` directory and entering it:
```sh
mkdir -p /usr/src
cd /usr/src
```

Download and extract the latest Linux kernel:

```sh
curl -fLO https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.7.9.tar.xz
tar xvf linux*
```

Now enter the directory and run `make menuconfig`:

```sh
cd linux*/
make menuconfig
```

This will open the kernel configuration TUI, where you can begin customizing your kernel:

![The kernel configuration screen.](1-menuconfig.png)

> For tips on how to configure the kernel, please see the Gentoo wiki's [kernel configuration guide.](https://wiki.gentoo.org/wiki/Kernel/Gentoo_Kernel_Configuration_Guide)

Once the kernel has been configured to your liking, compile and install it:

```sh
make -j$(nproc)
make install
```

You may also install the kernel modules.

```
make INSTALL_MOD_STRIP= 1 modules_install -j$(nproc)
```

Finally, be sure to give your kernel an appropriate name for the bootloader (`grub`) to identify:

```sh
mv /boot/vmlinuz-linux /boot/vmlinuz-linux-{{<hl>}}6.7.9{{</hl>}}
```

## Users and passwords

Set a password for the root user:

```sh
passwd
```

Create a new user:

```sh
useradd -m -s /bin/sh alex
```

## Bootloader configuration

To install `grub` bootloader, run the following commands:

```sh
grub-install {{<hl>}}/dev/sda{{</hl>}}
grub-mkconfig -o /boot/grub/grub.cfg
```

You should now be able to reboot and utilize your KISS Linux system.

Congratulations! You've successfully installed KISS Linux.

## Further Reading

- For more in-depth knowledge on KISS Linux and how to use its software, consult the [KISS Linux Community website.](https://kisscommunity.org/)

- To discover more software on KISS, consider using the [kiss find utility](https://github.com/aabacchus/kiss-find) or the [kiss find website.](https://jedahan.com/kiss-find/)

- To find more lists and repositories for KISS, consult the [awesome-kiss repository](https://github.com/kiss-community/awesome-kiss) on GitHub.



