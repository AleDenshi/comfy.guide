---
## Guide Information
title: "Gentoo Linux"
description: A minimalistic Linux distro made for tinkerers.
icon: gentoo.svg
date: 2024-02-10T22:28:57-05:00
#ports: [80, 443]

## Author Information
author: hdfsyu
#email:
#xmpp:
#matrix:
#monero:

draft: true
---
[Gentoo](gentoo.org) is a source based Linux Distro focused on versatility and minimalistic design. It's install is fully manual and runs on the OpenRC/systemd init system. It uses the `emerge/Portage` package manager/build system which compiles everything from source, including the kernel. This tutorial will only be focusing on: ***Setting up our computer for Linux, installing gentoo via the Stage 3 Tarball, installing the Linux kernel, and installing our necessary extra system tools.***
## Requirements
A USB/Flash drive which has [Linux Mint](linuxmint.com) or an existing Linux (preferably debian-based) distro install.

A basic knowledge of Linux and the command line.

**Recommended** A wired internet connection (Ethernet).
## Partitioning our hard drive
To get started with installing gentoo, switch to the root user using the `sudo su` command. We have now switched to the root user, we are going to run the `cfdisk` interactive disk partition utility.
### Creating our EFI boot partition.
Let's create our EFI boot drive, scroll down to the New button using your arrow keys and create a new 100MB partition, do this by writing 100M when cfdisk asks you for the size.
### Creating our Linux swap partition.
Now, let's create our swap drive. Do the same procedure as the EFI partition, but this time make the size double/the actual size of your PC's RAM.
### Creating our root partition.
This is the easiest, first do the same procedure, then press enter (Do not edit the size, keep it default). This will take all the empty space and create a parition using the remaining space of our drive.
Scroll down to write, type `yes` and exit cfdisk.
## Formatting our partitions
Run the `lsblk` command to list your block devices (hard drives), choose the appropriate hard drive. You can tell by looking at the `TYPE` and making sure it is disk, usually the biggest drive is your installation drive. Note down the name of your drive, usually `/dev/`*id* 
### Formatting our root partition
Run the following command to format our root partition as Ext4: `mkfs.ext4 /dev/`*id*, where *id* is your disk specifier.
### Formatting our EFI partition
Run the following command to format our EFI parition as FAT32: `mkfs.fat -F 32 /dev/`*id*.
### Formatting our swap partition
Run the following command to format our swap partition as swap: `mkswap /dev`*id*.
## Mounting our partitions
First, we'll need to create the `/mnt/gentoo` mountpoint, make the directory /mnt/gentoo using `mkdir /mnt/gentoo`.
### Mounting our root partition
Our root partition will be mounted at `/mnt/gentoo`, we can mount it at `/mnt/gentoo` using `mount /dev/`*root partition*` /mnt/gentoo`
### Mounting our EFI partition
Since we already have /mnt/gentoo, we will mount our boot partition at `/mnt/gentoo/boot/efi`. However, the boot nor the efi sub-directories have been made yet. We can create these directories using `mkdir -p /mnt/gentoo/boot/efi`, and then mounting our EFI parition using `mount /dev/`*efi partition*` /mnt/gentoo/boot/efi`
### Mounting/Activating our swap partition
This is the easiest step, just run `swapon /dev/*swap partition*`, and this will activate your swap partition.
We can run `lsblk` to see all our partitions mounted. Your swap partition should have [SWAP] next to it, your root partition should have /mnt/gentoo and your EFI partition should have /mnt/gentoo/boot/efi. We have now set up our PC for Linux, we are now going to install the base Gentoo files using the Stage 3 Tarball.
## Installing Gentoo Linux via the Stage 3 Tarball.
Since we are *not* using the gentoo minimal LiveCD, we can easily `wget` the tarball. But, we will need to configure our date for SSL/HTTPS downloads.
### Configuring the date & time
The `date` command shows us the current time, date and timezone. If these are wrong, set the date by using the following command, and subsitute the command with your date and time.

`date MMDDhhmmYYYY`

**M**onth, **d**ay, **h**our, **m**inute, and **y**ear.
### Downloading the tarball
Navigate to the [Gentoo Stage 3 Tarball downloads](https://www.gentoo.org/downloads/), and choose your preferred stage archive. In this tutorial, we will be going with a base, minimal installation of Gentoo. If you are planning to use a desktop, choose the desktop profile | openrc option. In this tutorial, we will be choosing the Stage 3 openrc tarball, for a minimal installation with the OpenRC init system. We are now going to download the tarball, *do not download the tarball*, instead we will copy the link by right clicking, then pressing the Copy Link button, and going into our terminal. We now have the link on our clipboard, change directory to the `/mnt/gentoo` directory using `cd /mnt/gentoo`. Then, run the following command: `wget LINKTOTARBALLHERE`. In this tutorial, I will use `wget https://distfiles.gentoo.org/releases/amd64/autobuilds/20240204T134829Z/stage3-amd64-openrc-20240204T134829Z.tar.xz` to download the base stage 3 tarball.
### Extracting the tarball
Run `tar xpvf stage3-*.tar.xz --xattrs-include='*.*' --numeric-owner` and wait for the tarball to extract, this command extracts and preserves permissions while extracting the tarball.
## Configuring the Portage build system
You must configure the portage `make.conf` file to configure your make options. Run `nano /mnt/gentoo/etc/portage/make.conf` to edit this file, and add this line under FFLAGS:

`MAKEOPTS="-j5 -l5"`. In this case I have 10GB of RAM and five CPU threads. So I will use the -j5 and -l5 options. Now edit your COMMON_FLAGS variable in your portage config, to this:

`COMMON_FLAGS="-O2 -pipe -march=native"`. This sets the CPU architecture to whatever family processor you have, which makes the binaries tailored to your system only. Press CTRL+S and CTRL+X to exit the `nano` text editor.
### Changing our Gentoo mirrors
Navigate to the [Gentoo source mirror list](https://www.gentoo.org/downloads/mirrors/) and find the mirror that is closest to you. Copy the link and edit the portage make.conf by running `nano /mnt/gentoo/etc/portage/make.conf`. Then, at the bottom of the file, add:

`GENTOO_MIRRORS="LINK TO MIRROR"`

This will make the installation process *way* faster.

This doesn't really have a category, but quickly copy your DNS info to the gentoo partition by using `cp --dereference /etc/resolv.conf /mnt/gentoo/etc/`.

## Entering the Gentoo root file system.
We are now entering our newly installed system.
### Mounting all our file systems
We are going to mount our `/proc/, /sys, /dev/ and /run` file systems using a list of commands. These list of commands are:

- `mount --types proc /proc /mnt/gentoo/proc`
- `mount --rbind /sys /mnt/gentoo/sys`
- `mount --make-rslave /mnt/gentoo/sys`
- `mount --rbind /dev /mnt/gentoo/dev`
- `mount --make-rslave /mnt/gentoo/dev`
- `mount --bind /run /mnt/gentoo/run`
- `mount --make-slave /mnt/gentoo/run`

Just copy and paste those commands into your terminal.

### Fixing /dev/shm's symbolic link
There is a /run/shm symbolic link in Linux Mint which points to /dev/shm, we will need to remove this. Run these list of commands:

- `test -L /dev/shm && rm /dev/shm && mkdir /dev/shm`
- `mount --types tmpfs --options nosuid,nodev,noexec shm /dev/shm`

Set permissions on /dev/shm to 1777 using this command: `chmod 1777 /dev/shm`

### Entering our new Gentoo installation
Just run the `chroot /mnt/gentoo /bin/bash` which changes our directory to our new Gentoo installation using the bash terminal. Source /etc/profile using `source /etc/profile`