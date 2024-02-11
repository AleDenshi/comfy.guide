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
[Gentoo](gentoo.org) is a source based Linux Distro focused on versatility and minimalistic design. It's install is fully manual and runs on the OpenRC/systemd init system. It uses the `emerge` package manager which compiles everything from source, including the kernel. This tutorial will only be focusing on: ***Setting up our computer for Linux, installing gentoo via the Stage 3 Tarball, installing the Linux kernel, and installing our necessary extra system tools.***
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
Since we are *not* using the gentoo minimal LiveCD, we can easily `wget` the ISO file. But, we will need to configure our date for SSL/HTTPS downloads.
### Configuring the date & time
The `date` command shows us the current time, date and timezone. If these are wrong, set the date by using the following command, and subsitute 