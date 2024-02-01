---
## Guide Information
title: "Arch"
description: "description"
icon: archlinux.svg
date: 2024-02-01T17:37:15-04:00
## ports: [80, 443]

## **Contents**   

---
 
# Arch Linux Installation Guide <a name='id0'></a>

Welcome to this guide to install Arch Linux on your computer. This tutorial is rooted in the official Arch Linux documentation, to ensure that you’re receiving accurate and up-to-date information.

In addition to the standard procedures, this guide introduces alternative commands and strategies designed to streamline the installation process and cater to your unique preferences. Whether you’re a seasoned Linux user or a newcomer to the world of open-source operating systems, you’ll find these modifications helpful in personalizing your Arch Linux experience.

This repository contains various configuration tools that can assist you. However, this guide will focus on installing the BSPWM desktop environment. If you prefer to use a different graphical environment, please refer to the official Arch Wiki, which provides comprehensive guides for installing various [desktop environments](https://wiki.archlinux.org/title/Desktop_environment).

For further information and support, the official Arch Linux forum is an invaluable resource. Here, you’ll find the official [**Installation Guide**](https://wiki.archlinux.org/title/installation_guide), along with a wealth of knowledge from the Arch Linux community.

# 1) Initial Configurations <a name='id1'></a>

The upcoming section focuses on initializing Arch Linux prior to its installation on the system. In this stage, you will be required to execute a series of commands to configure your keyboard and establish an internet connection, among other crucial settings. The successful execution of these commands is vital for a smooth installation process and to prevent potential issues in the future.

## 1.1) Network Configuration <a name='id1.1'></a>

***Note**: If the computer on which you are going to install Arch is connected by cable, you can skip the following part, as it is the configuration of the wireless connection.*

We are gonna use the tool `iwctl` will be used for the internet configuration

    $ iwctl

After executing the command you have to look for the technical name of your wifi card with the command `device list`.

    $ device list

The name of your wifi card will be the one you will place in the **wlan** section.

    $ station <wlan> connect <Network Name>

***Note**: If your network is hidden, you must replace the `connect` with the `connect-hidden` attribute.*

After this, it is advisable to test the connection with the `ping` command.

# 2) Pre-Installation <a name='id2'></a>

If you want a simple installation, you can use **archinstall**, however this is not 100% reliable and I recommend installing it manually.

## 2.1) Partitioning disk <a name='id2.1'></a>

The first step is to identify the path of the partition we want to manage. We can do this by using the `fdisk -l` command, which lists all the disks and their partitions.

When you run `fdisk -l`, look for your disk in the output. You can identify your disk based on its size or model. For instance, in my case, I have an NVMe drive, so it appears as `nvme0n1`.

Once you've identified your disk, you can use the `cfdisk` command followed by the path to your disk. In my case, the command would be 

    $ cfdisk /dev/nvme0n1

We will be using the `cfdisk` tool to partition the disk into three sections: boot, swap, and root. It is advisable to use the **gpt** label type, as it is prevalent in UEFI systems. If you have partitions already created from a previous operating system, you will need to delete all of them. 

- **The boot partition**: It is recommended to allocate 100M for the boot partition. This partition is essential for the system to boot up.
- **The swap partition**: The size of the swap partition should be a power of 2 (2, 4, 8, 16, etc.), depending on the size of your hard drive. In this case, it is recommended that the swap partition be at least 8GB. The swap partition acts as an overflow for your system memory, ensuring smooth operation when your RAM is fully utilized.
- **The root partition**: Allocate the remaining hard drive space to the root partition. This partition will contain your operating system, applications, and files.

Once you have partitioned the disk, write the changes and exit the `cfdisk` tool.

To list the partitions and track your progress, use the `lsblk` command. This command is crucial for confirming the ID, size, and type of the partitions.

## 2.2) Formatting the Partitions <a name='id2.2'></a>

In this step, we will format the three partitions that we have created. 

1. **Root Partition**: The first partition we need to format is the root partition. This can be accomplished using the command below:

        $ mkfs.ext4 /dev/nvme0n1p3

    This command forma1ts the partition as an ext4 filesystem, which is a common choice for Linux installations due to its robustness and excellent performance.

2. **Boot Partition**: Next, we will format the boot partition. The boot partition is crucial for the system startup process. Use the following command to format it:

        $ mkfs.fat -F 32 /dev/nvme0n1p1

    This command formats the partition with a FAT32 filesystem. FAT32 is commonly used for boot partitions as it is universally supported by almost all operating systems.

3. **Swap Partition**: Finally, we will set up the swap partition. The swap partition is used as a 'backup' for your system's physical memory, providing extra resources if your system runs out of RAM. Use the following command to format it:

        $ mkswap /dev/nvme0n1p2

    This command initializes the partition to be used as swap space.

***Note:** Remember to replace `/dev/nvme0n1pX` with your actual partition paths if they are different. Always double-check your commands before executing them to avoid data loss.* 

## 2.3) Mounting the Partitions <a name='id2.3'></a>

In this step, we will be mounting the partitions. First, let's start with the **root** partition. You can mount it using the command below:

    $ mount /dev/nvme0n1p3 /mnt

Next, we need to mount the **boot** partition. However, the required path does not exist yet. Therefore, we will create it using the following command:

    $ mkdir -p /mnt/boot/efi

With the path now created, we can proceed to mount the **boot** partition:

    $ mount /dev/nvme0n1p1 /mnt/boot/efi

Lastly, the **swap** partition does not need to be mounted in the traditional sense. Instead, it needs to be activated. You can do this with the following command:

    $ swapon /dev/nvme0n1p2

# 3) Installation <a name='id3'></a>

## 3.1) Basic packages <a name='id3.1'></a>

The installation process involves selecting the desired packages and mounting them in the `/mnt` directory. It is recommended to install at least the following packages: `base`, `linux`, `linux-firmware`, `base-devel`, `grub`, `efibootmgr`, `nano`, `networkmanager`, `git`, `pulseaudio` and `intel-ucode`.

***Note**: For those using an AMD processor, it's necessary to install the `amd-ucode` package instead of `intel-ucode`.*

To install these packages, use the command below:

    $ pacstrap /mnt base linux linux-firmware base-devel grub efibootmgr nano networkmanager git pulseaudio intel-ucode

This command will install the base system along with the Linux kernel and firmware, development tools, the GRUB bootloader, EFI boot manager, a basic text editor (nano), network manager, Git for version control, and microcode for Intel processors. Remember to replace `intel-ucode` with `amd-ucode` if you're using an AMD processor. This will ensure your system has the latest microcode updates from AMD. 

## 3.2) File System Tab <a name='id3.2'></a>

Once you've installed the necessary tools, the next step is to generate a **fstab** file. This file is crucial as it allows your system to automatically mount partitions upon booting. 

You can generate a **fstab** file using the following command:

    $ genfstab /mnt

This command will display information about the currently mounted files. However, you need to transfer this information to disk. To do this, you can redirect the output of the `genfstab` command to the **fstab** file located in the `/mnt/etc/` directory:

    $ genfstab /mnt > /mnt/etc/fstab

To ensure that the **fstab** file has been correctly generated, you can use the `cat` command to display its contents:

    $ cat /mnt/etc/fstab

The output should match the initial output of the `genfstab /mnt` command. If it does, then you've successfully generated your **fstab** file and are ready to proceed to the next step of the installation process.

## 3.3) Switching to the Installed System (Changing Root) <a name='id3.3'></a>

In this step, we will transition into our newly installed system environment. To do this, we use the following command:

    $ arch-chroot /mnt

# 4) Internal Configuration <a name='id4'></a>

## 4.1) Setting the Time Zone <a name='id4.1'></a>

The first step in our internal configuration process is to set the system's time zone. This can be done by creating a symbolic link between our desired time zone and `/etc/localtime`. Replace **Continent** and **Country** with your specific location. After setting the time zone, we will check the system date and synchronize the hardware clock with the system clock. The commands are as follows:

    $ ln -sf /usr/share/zoneinfo/Continent/City /etc/localtime
    $ date
    $ hwclock --systohc

## 4.2) Configuring Localization <a name='id4.2'></a>

Next, we will configure the system's locale settings. This involves editing the `locale.gen` file to uncomment the line corresponding to our desired locale. In this case, we will be using `en_US.UTF-8 UTF-8`. After saving the changes, we generate the new locale configuration using the `locale-gen` command:

    $ nano /etc/locale.gen
    # Uncomment the line: en_US.UTF-8 UTF-8
    $ locale-gen

For certain programs to function correctly, we need to specify our locale in the `/etc/locale.conf` file. We can do this by adding the line `LANG=en_US.UTF-8` to the file. Here's the command to do it:

    echo LANG=en_US.UTF-8 > /etc/locale.conf

This command writes `LANG=en_US.UTF-8` to the `/etc/locale.conf` file. Now, your system-wide locale is set and will be recognized by all locale-aware programs on your system. Remember to replace `en_US.UTF-8` with your desired locale if it's different. 

## 4.3) Configuring the Keyboard Layout (Keymap) <a name='id4.3'></a>

If you're using an English keyboard, this step may not be necessary. However, if you need to change the keyboard layout, you can do so by modifying the `/etc/vconsole.conf` file. 

To set the keyboard layout to US English, add the following line to the file:

    $ echo KEYMAP=us > /etc/vconsole.conf

If you want to use a variant of the US layout, such as the international layout, you would add it like this:

    $ echo KEYMAP=us-acentos > /etc/vconsole.conf

Please replace `us-acentos` with your desired keymap. This command writes `KEYMAP=us-acentos` to the `/etc/vconsole.conf` file. Now, your system-wide keymap is set and will be recognized by your system.

## 4.4) Setting the Hostname <a name='id4.4'></a>

The hostname is a crucial aspect of your system configuration because it determines the internal name of your computer. To set the hostname, you need to access the file located at `/etc/hostname` and enter your desired name there. Here's how you can do it:

    $ echo YourDesiredHostname > /etc/hostname

Replace 'YourDesiredHostname' with the name you want to assign to your computer.

## 4.5) Setting the Root Password <a name='id.5'></a>

Setting the root password is a straightforward process, but it's vital for your system's security. The root password is what you'll use every time you need to perform tasks with root privileges, so it should be complex to prevent unauthorized access. 

You can set the root password using the `passwd` command. After entering the command, you'll be prompted to type your password twice to confirm it. Here's how you can do it:

    $ passwd
    # You'll be prompted to type your password twice

Remember, a strong password typically includes a mix of upper and lower case letters, numbers, and special characters.

## 4.6) Creating a New User <a name='id4.6'></a>

Firstly, we will use the `useradd` command with the `-m` flag, which instructs the system to create a `/home` directory for the new account. The `-G` option is used to specify the group that should own the user’s home directory, in this case, `wheel`. The `-s` option sets the default shell for the user, which we will set to `/bin/bash`. Replace '(name)' with the desired username. 

    $ useradd -m -G wheel -s /bin/bash (name)
    $ passwd (name)

Next, we will set up sudo for the new user. As it stands, if we switch to our new user using the `su (user)` command and attempt to execute a command with sudo (for example, `sudo pacman -Syu`), we will encounter an error after entering our password. 

To rectify this, we need to exit our current user session using either the `exit` command or `sudo su`. Then, we will open the sudoers file using the **visudo** command with our preferred editor set by the **EDITOR** environment variable:

    $ EDITOR=nano visudo

In the sudoers file, locate and uncomment the line `%wheel ALL=(ALL) ALL`. This grants all members of the **wheel** group full sudo privileges. 

Now, if we switch back to our new user and attempt to use sudo commands, we should be able to do so without encountering any errors.

## 4.7) Enabling Network Manager <a name='id4.7'></a>

To ensure that your system can connect to the internet, you'll need to enable the Network Manager. This can be done by running the following command in the terminal:

    $ systemctl enable NetworkManager
    $ systemctl enable NetworkManager.service

## 4.8) Installing the Bootloader <a name='id4.8'></a>

The next step, which is arguably the most crucial, involves installing a bootloader. Without a bootloader, your system will not be able to start. In this guide, we'll be using GRUB as our bootloader. To install GRUB, execute the following command:

    $ grub-install /dev/nvme0n1

After installing GRUB, it needs to be configured. This can be accomplished with the following command:

    $ grub-mkconfig -o /boot/grub/grub.cfg

## 4.9) Final Steps and Rebooting the System <a name='id4.9'></a>

Once GRUB has been configured, you can exit the root user, unmount all mounted filesystems, and reboot your system. This can be done by running the following commands:

    $ exit
    $ umount -a
    $ reboot

After rebooting, your Arch Linux installation should be complete and ready to use. Enjoy exploring your new system!

# 5) Post-Installation Tasks <a name='id5'></a>

## 5.1) Establishing the Internet Connection <a name='id5.1'></a>

Once the system is installed, it's recommended to retest the internet connection. This can be done using the `ping` command. 

    $ ping -c 3 www.google.com

If you're unable to establish an internet connection, the `nmcli` command will be your go-to solution. This command allows you to manage NetworkManager and any associated network connections.

To add a new connection, you can use the following command:

    $ nmcli c add type wifi con-name <connect name> ifname <wlan> ssid <ssid>

***Note**: The `connect name` is a customizable identifier that you can assign to your network. This name is not fixed and can be changed according to your preference.*

This command creates a new connection with the type `wifi`. The `<connect name>` is the name you assign to the connection, `<wlan>` is the interface name, and `<ssid>` is the SSID of the wireless network.

To connect to a hidden wireless network, use:

    $ nmcli dev wifi connect <ssid> password <password> hidden yes

This command allows you to connect to a hidden network by specifying the SSID and password.

If you need to delete a connection, you can do so with:

    $ nmcli c delete <connect name>

This command deletes the network connection associated with the specified 'connect name'.

Sure, I'd be happy to help you improve and expand your guide. Here's a revised version of your text:

## 5.2) Configuring the DNS  <a name='id5.2'></a>

One of the crucial steps in setting up your internet connection is configuring the Domain Name System (DNS). This step is important to ensure seamless connectivity and to avoid potential issues, such as those that might occur with Microsoft services. 

To begin, you need to identify the name of your connection. This can be done by executing the following command in your terminal:

    $ nmcli con

This command will list all your active connections. Identify the connection for which you want to set the DNS.

Once you have the name of your connection (referred to as `<ssid>`), you can modify its DNS settings. Google's DNS servers (`8.8.8.8` and `8.8.4.4`) are commonly used due to their reliability. To set these as your DNS servers, use the following command:

    $ nmcli con mod "<ssid>" ipv4.dns "8.8.8.8 8.8.4.4"

Replace `<ssid>` with the name of your connection. This command sets the DNS servers for your specified connection to Google's DNS servers.

## 5.3) Battery Optimization <a name='id5.3'></a>

If you're installing Arch Linux on a laptop, optimizing battery life is crucial. One effective tool for this purpose is `auto-cpufreq`. This utility dynamically adjusts the frequency of your CPU based on load and power source. Here's how you can install and use it:

First, clone the repository from GitHub:

    $ git clone https://github.com/AdnanHodzic/auto-cpufreq.git

Next, navigate to the cloned directory and run the installer:

    $ cd auto-cpufreq && sudo ./auto-cpufreq-installer

Once the installation is complete, you need to activate `auto-cpufreq`. You can do this by running the following command:

    $ sudo auto-cpufreq --install

Remember, `auto-cpufreq` requires root privileges to make changes to your system. Always be cautious when using `sudo` with any command.

With `auto-cpufreq` installed and active, your laptop should now be better optimized for battery life. For more detailed information about your system's performance, you can use the `auto-cpufreq --stats` command to display real-time statistics.

## 5.4) Extra Configurations <a name='id5.4'></a>

In the `/etc/pacman.conf` file, I highly recommend making a few adjustments to enhance your experience. First, find the line that reads `Color` and uncomment it. This will enable colored output, making it easier to read and understand the information displayed in your terminal. 

Next, look for `ParallelDownloads` and set its value to 5. This allows for multiple packages to be downloaded simultaneously, speeding up the installation process.

Finally, uncomment the `ILoveCandy` line. While this doesn't impact the functionality, it does replace the standard download progress bar with a fun, candy-themed one. It's a small touch, but it adds a bit of whimsy to your Arch Linux setup process.

The subsequent steps largely depend on the user's preferences, but it's generally advisable to set up a graphical environment for ease of use.

Remember, the beauty of Arch Linux lies in its flexibility. You can customize your system to suit your preferences. Enjoy the journey of making Arch Linux your own!

## Author Information
author: Druxorey

---

