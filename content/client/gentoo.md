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

draft: false
---
[Gentoo](https://gentoo.org) is a source based Linux Distro focused on versatility and minimalistic design. It's install is fully manual and runs on the OpenRC/systemd init system. It uses the `emerge/Portage` package manager/build system which compiles everything from source, including the kernel. This tutorial will only be focusing on: ***Setting up our computer for Linux, installing gentoo via the Stage 3 Tarball, installing the Linux kernel, and installing our necessary extra system tools.***
## Requirements
A USB/Flash drive which has [Linux Mint](https://linuxmint.com) or an existing Linux (preferably debian-based) distro install.

A basic knowledge of Linux and the command line.

An internet connection.

**Recommended** A *wired* internet connection (Ethernet).
## Partitioning our hard drive
To get started with installing gentoo, switch to the root user using the `sudo su` command. We have now switched to the root user, we are going to run the `cfdisk` interactive disk partition utility.

Delete all the partitions that you have right now using the Delete option.
### Creating our EFI boot partition.
Let's create our EFI boot drive, scroll down to the New button using your arrow keys and create a new 512MB partition, do this by writing 512M when cfdisk asks you for the size.
### Creating our Linux swap partition.
Now, let's create our swap drive. Do the same procedure as the EFI partition, but this time make the size double/the actual size of your PC's RAM.
### Creating our root partition.
This is the easiest, first do the same procedure, then press enter (Do not edit the size, keep it default). This will take all the empty space and create a parition using the remaining space of our drive.
Scroll down to write, type `yes` and quit cfdisk.
## Formatting our partitions
Run the `lsblk` command to list your block devices (hard drives), choose the appropriate hard drive. You can tell by looking at the `TYPE` and making sure it is disk, usually the biggest drive is your installation drive. Note down the name of your drive, usually `/dev/`*id*
### Formatting our root partition
Run the following command to format our root partition as Ext4: `mkfs.ext4 /dev/`*root*, where *root*, *efi* or *swap* is your disk specifier.
### Formatting our EFI partition
Run the following command to format our EFI parition as FAT32: `mkfs.fat -F 32 /dev/`*efi*.
### Formatting our swap partition
Run the following command to format our swap partition as swap: `mkswap /dev`*swap*.
## Mounting our partitions
First, we'll need to create the `/mnt/gentoo` mountpoint, make the directory /mnt/gentoo using `mkdir /mnt/gentoo`.
### Mounting our root partition
Our root partition will be mounted at `/mnt/gentoo`, we can mount it at `/mnt/gentoo` using `mount /dev/`*root partition*` /mnt/gentoo`
### Mounting our EFI partition
Since we already have /mnt/gentoo, we will mount our boot partition at `/mnt/gentoo/boot/efi`. However, the boot nor the efi sub-directories have been made yet. We can create these directories using `mkdir -p /mnt/gentoo/boot/efi`, and then mounting our EFI parition using `mount /dev/`*efi partition*` /mnt/gentoo/boot/efi`
### Mounting/Activating our swap partition
This is the easiest step, just run `swapon /dev/*swap partition*`, and this will activate your swap partition.
We can run `lsblk` to see all our partitions mounted. Your swap partition should have [SWAP] next to it, your root partition should have /mnt/gentoo and your EFI partition should have /mnt/gentoo/boot/efi. We now have to set up our PC for Linux, we are now going to install the base Gentoo files using the Stage 3 Tarball.
## Installing Gentoo Linux via the Stage 3 Tarball.
Since we are *not* using the gentoo minimal LiveCD, we can easily `wget` the tarball. But, we will need to configure our date for SSL/HTTPS downloads.
### Configuring the date & time
The `date` command shows us the current time, date and timezone. If these values are wrong, set the date by using the following command, and subsitute the command with your date and time.

`date MMDDhhmmYYYY`

**M**onth, **d**ay, **h**our, **m**inute, and **y**ear.
### Downloading the tarball
Navigate to the [Gentoo Stage 3 Tarball downloads](https://www.gentoo.org/downloads/), and choose your preferred stage archive. In this tutorial, we will be going with a base, minimal installation of Gentoo. If you are planning to use a desktop, choose the desktop profile | openrc option. In this tutorial, we will be choosing the Stage 3 openrc tarball, for a minimal installation with the OpenRC init system. We are now going to download the tarball, *do not download the tarball*, instead we will copy the link by right clicking, then pressing the Copy Link button, and going into our terminal. We now have the link on our clipboard, change directory to the `/mnt/gentoo` directory using `cd /mnt/gentoo`. Then, run the following command: `wget LINKTOTARBALLHERE`. In this tutorial, I will use `wget https://distfiles.gentoo.org/releases/amd64/autobuilds/20240204T134829Z/stage3-amd64-openrc-20240204T134829Z.tar.xz` to download the base stage 3 tarball.
### Extracting the tarball
Run `tar xpvf stage3-*.tar.xz --xattrs-include='*.*' --numeric-owner` and wait for the tarball to extract, this command extracts and preserves permissions while extracting the tarball.
## Configuring the Portage build system
You must configure the portage `make.conf` file to configure your make options. Run `nano /mnt/gentoo/etc/portage/make.conf` to edit this file, and add this line under FFLAGS:

`MAKEOPTS="-j5 -l5"`. In this case I have 10GB of RAM and five CPU threads. So I will use the -j5 and -l5 options. Now edit your COMMON_FLAGS variable in your portage config, to this:

`COMMON_FLAGS="-O2 -pipe -march=native"`. This sets the CPU architecture to whatever family processor you have, which makes the binaries tailored to your system only. Press CTRL+S and CTRL+X to exit the `nano` text editor. Remember the CTRL+S and CTRL+X part, we'll be using that to save and exit nano.
### Changing our Gentoo mirrors
Navigate to the [Gentoo source mirror list](https://www.gentoo.org/downloads/mirrors/) and find the mirror that is closest to you. Copy the link and edit the portage make.conf by running `nano /mnt/gentoo/etc/portage/make.conf`. Then, under MAKEOPTS add:

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
Just run the `chroot /mnt/gentoo /bin/bash` which changes our directory to our new Gentoo installation using the bash terminal. Source /etc/profile to import the environment variables using `source /etc/profile`, to make our command line look a bit prettier (and to avoid any mistakes) we'll add a (chroot) to our shell, run `export PS1="(chroot) ${PS1}"` to do this.

## Configuring our system to our needs and updating it
### Sync Gentoo mirrors
Run `emerge-webrsync` to sync the repository mirrors. This might take a minute.

### Select installation profile
There are multiple "installation profiles", and we'll need to select one of them to install the required packages. Do this by running `eselect profile list`, and looking at the profile list. For this tutorial, I will be sticking with the `default/linux/amd64/17.1` option that is chosen by default. You might be installing a desktop environment with your Gentoo system, so you can choose the `default/linux/amd64/17.1/desktop/kde` option, as the time of writing the option number is four.
In order to choose your profile, run `eselect profile choose PROFILENUMBERHERE` and replace PROFILENUMBERHERE with your desired profile's number.

### Configuring our USE flags/VIDEO_CARDS variable (OPTIONAL)
This is the part of Gentoo where we actually make it minimal and change it to our needs. First, we'll list our USE flags, and configure our VIDEO_CARDS variable. Run the command `emerge --info | grep ^USE` which lists the USE variables. Copy and paste the output of this command into your portage make.conf, `nano /etc/portage/make.conf` to open the file. After you have pasted it, take a look at the USE flags and modify it to your needs. You can add a `-` before a dependency to exclude it. Then, modify your VIDEO_CARDS variable, it should look something like:

`VIDEO_CARDS="blah blah blah blah"`

Remove some properties of the video cards variable to fit your PC, for example I have an AMD gpu, my VIDEO_CARDS variable would look like this:

`VIDEO_CARDS="amdgpu fbdev"`

Ok! You've either skipped this part (eww bloat) or you went through looking at each dependency and remove stuff. Let's... ***run the most infamous Gentoo command***.
### Configuring the ACCEPT_LICENSE variable
Oh yeah! Before we do that let's set the ACCEPT_LICENSE variable. Edit your portage make.conf for (hopefully the last time) by running `nano /etc/portage/make.conf`. Now, write `ACCEPT_LICENSE="*"` right above the `GENTOO_MIRRORS` variable.

Make sure to save your changes in nano!
### Updating the @world set
This is the most time consuming part, but first, let me explain what the @world set is. The @world set is a collection of packages inherited from your USE flags, for example if I have `plasma` in my USE flags, @world will contain `plasma`. Ok, let's update the @world set. Run `emerge --ask --verbose --update --deep --newuse @world` to update the @world set. This might take a day or 2 *if* you have a desktop profile with a 4/8 core system
### Cleaning the gunk off our system
Let's remove all the extra stuff we don't need off our system, run `emerge --ask --depclean` to clean the extra dependencies.
## Configuring our system
### Setting your timezone
Let's set our timezone so we can visit HTTPS secured websites without our browser complaining. In this tutorial, I will assume you already know your UNIX timezone, just run `echo "Region/City" > /etc/timezone`. In this case, I will run `echo "Asia/Dubai" > /etc/timezone`. Let's tell Gentoo about these changes.

Run `emerge --config sys-libs/timezone-data`, and now your timezone is set!
### Setting your locale
We now have to set our locale, run `nano /etc/locale.gen` and you will be prompted with a file.
Add your locale into the file, in this case I want a US locale, so I will write `en_US.UTF-8 UTF-8` in the file. Save and exit nano, then run `locale-gen` to generate our new locales. Now, we will *set* our locales. Run `eselect locale list`, and find your desired locale. In this case my locale is `en_US.utf8` with number four. Now, run `eselect locale set NUMBER`, in this case I will run `eselect locale set 4`.

Let's make our terminal reflect the changes, run this one-liner: `env-update && source /etc/profile && export PS1="(chroot) ${PS1}"`
## Configuring the Linux kernel
Now, we will actually install the kernel which will allow our hardware to interface with the apps.
### Configuring the kernel from source
This is possible, but ***very*** tedious and time consuming. In this tutorial, we'll use a binary kernel. A binary kernel is more bloated, but *way* easier to set up.
### Downloading the binary kernel
We're not even compiling the kernel, so this will take very minimal time.

First, we'll need to download the kernel binary. Run `emerge --ask sys-kernel/installkernel-gentoo` to install the `installkernel` command, this command will help up install the kernel automatically. Now, we'll download the precompiled kernel image using emerge, run `emerge --ask sys-kernel/gentoo-kernel-bin` to install the binary kernel.
### Installing linux-firmware
Now, we are going to install drivers for our hardware. We can quickly do this by running `emerge --ask sys-kernel/linux-firmware`.
## Finalizing and adding extra packages to our system
We're almost done! Congratulations, you made it this far.
### Generating the fstab
We are now generating a file called `fstab` or, "fs tab". This file is very important, it tells Linux where to mount our *root, swap and boot* partitions. But we aren't going to write it ourselves, we will use a script. Open a ***new*** terminal and do not close the other one. Clone the genfstab script from github using `git`. Run `git clone https://github.com/glacion/genfstab && cd genfstab`. In your new terminal, change to root. Do this by running `sudo su`, now run `chmod +x genfstab` to make the script executable. Finally, run
`./genfstab /mnt/gentoo > /mnt/gentoo/etc/fstab`. Close this terminal, and go back to your terminal with the (chroot) in front of it. ***Make sure you are in the Gentoo chroot environment***
#### Editing the fstab a bit
Run nano and edit /etc/fstab, run `nano /etc/fstab`. The three lines saying `efivarfs`, `none` and `tracefs`, put a # at the beginning of those lines.
### Creating our hostname
Let's quickly create our hostname, all we need to do is write our hostname in the `/etc/hostname` file. Do this by running this command: `echo "YOURHOSTNAME" > /etc/hostname`. This will write whatever `echo` outputs to /etc/hostname, in this case "YOURHOSTNAME". I want to set my hostname to `gentooey`, so I will run `echo "gentooey" > /etc/hostname`. Run `cat /etc/hostname` to confirm your hostname is in the file.
### Change the hosts file
We are going to edit the `/etc/hosts` file really quickly, just adding one line. Run nano using this command and edit the /etc/hosts file: `nano /etc/hosts`. Above the line that says `127.0.0.1   localhost`, write

`127.0.0.1   YOURHOSTNAME`. Since my hostname is `gentooey`, I will write

`127.0.0.1    gentooey`.
### Setting our root password
Now we will add a bit of security to our install using passwords. Just run the `passwd` command and type your password twice, once for input and twice for confirmation.
### Installing more required packages
Just run `emerge --ask networkmanager sudo efibootmgr grub`. This will install our bootloader, which is GRUB, enable support for EFI and internet support. Run this command.
### Installing our bootloader
The bootloader is what loads our system. First, let's install the bootloader, then configure it. Run `grub-install /dev/`*id*. In this case my device id is `sda` so I will run `grub-install /dev/sda`. Now, let's *configure* the GRUB bootloader.

We can automatically generate this, just run `grub-mkconfig -o /boot/grub/grub.cfg` to generate the GRUB config.
### Making our user account
Let's make our new user. There is a utility called `useradd` which we can use to make our user account. Just run these commands, and replace YOURUSERNAME with your desired username.

- `useradd -m -G audio,cdrom,floppy,portage,usb,video,wheel -s /bin/bash YOURUSERNAME`

- `passwd YOURUSERNAME`

### Making our user administrator
Let's make our user a ***SUPER USER***! We will need to edit our sudoers file, run `EDITOR=nano visudo` to set the preferred editor to nano, and edit the sudoers file.

Scroll down until you see `# %wheel ALL=(ALL) ALL`.
Remove the # at the beginning of the line.
### The last step (Cleaning our drive and rebooting)
Navigate to the root directory in our chroot, just run `cd /`. Now, let's clean up our downloaded stage 3 tarball. Run `rm /stage3*`.

And finally...

Type `exit` and press enter in your terminal...

and run the command `reboot`.

## Congratulations!
You now have a working Gentoo system! Use the nmtui to connect to networks, and the `emerge` command to install your favorite packages! You have now installed Gentoo Linux.
