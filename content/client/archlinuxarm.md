---
## Guide Information
title: "Arch Linux ARM"
description: How to install Arch Linux ARM, with additional steps for installing on a Raspberry Pi 4/5.
icon: archlinux.svg
date: 2024-02-02T09:20:31+10:00
draft: true
## Author Information
author: zacoons
email: zac@zacoons.com
matrix: "@zac:zacoons.com"
---

Regular Arch Linux is only available for x86 devices, so if you want Arch on your Raspberry Pi you must use Arch Linux ARM.

Arch Linux ARM doesn't have an ISO image, it only has a tarball, so we must format the hard drive and unpack the OS manually.

> When I refer to "the hard drive" I am referring to the hard drive of your ARM device. Be very careful not to touch the hard drive of your PC.

## Extra Work For Windows People (skip if you're running Linux)

Hello Windows user. Unfortunately, you must install a virtual machine so you can run Linux to format the hard drive. Trust me, I have tried to do this without Linux and it is impossible. Windows doesn't support formatting disks with EXT filesystems. If you figure out a way to do it on Windows, please edit this guide.

### Step 1, Downloading Requirements

Go to the [VirtualBox downloads page](https://www.virtualbox.org/wiki/Downloads) and select "Windows hosts", then run the installer.

Go to the [Arch Linux downloads page](https://archlinux.org/download/) and click "Magnet link for &lt;version number> 🧲". Click the "Start Torrent" button, then "Save file" and save it to your "Downloads" directory.

### Step 2, Setting Up the Virtal Machine (VM)

> When I tell you to open VirtualBox or Command Prompt, be sure to open it with administrator privileges.

Open VirtualBox and click "New".\
Name it "archie" or something,\
click on the "ISO Image" dropdown, select "Other...", and select the Arch Linux ISO in your "Downloads" directory,\
then click "Next", "Next", "Next", and "Finish".

Now go to "Settings > Storage" on your virtual machine and click the rightmost plus button on "Controller: SATA",\
enable "Use Host I/O Cache",\
then click "Ok", and exit VirtualBox.

### Step 3, Connecting the Hard Drive To the VM

> Make sure VirtualBox is closed for this step.

Plugin in the hard drive to your PC. Then open Command Prompt and type the following commands. They will allow you to access your hard drive from VirtualBox.

```
wmic diskdrive list brief
```

Now figure out the name of your hard drive. It's probably going to be `\\.\PHYSICALDRIVE1`.

```
cd "C:\Program Files\Oracle\VirtualBox"
VBoxManage.exe internalcommands createrawvmdk -filename C:\harddrive.vmdk -rawdisk <hard drive name>
```

Make sure to replace `<hard drive name>` with the name of the hard drive, e.g. `\\.\PHYSICALDRIVE1`.

> Before you do this, be certain that it is the hard drive <u>of your ARM device</u> and not the hard drive of your PC. <u>You will be wiping the drive in future steps.</u>

Now open VirtualBox and again navigate to "Settings > Storage".

Click the rightmost plus button on "Controller: SATA", select "armharddrive.vmdk", and hit "Choose".

Now run the VM and install wget. You'll need to mount the main storage of your VM, which will probably be called `sda`. But be sure to check with `lsblk`.

```
pacman-key --init
pacman -S wget

mkdir sda
mkfs.ext4 /dev/sda
mount /dev/sda sda
cd sda
```

## Formatting the Hard Drive

Make sure the hard drive is plugged in to your PC.

Run the command `lsblk` to list the connected hard drives and locate the hard drive of your ARM device. It will probably be called `sdb`.

You can format it with `cfdisk` or `fdisk`. I will only be showing you how to use `fdisk`, but `cfdisk` is more intuitive.

If you wish to use `cfdisk` or any other partitioning tool, you'll need the following partitions:
1. FAT32 (LBA) of size 300M
2. (optionally) SWAP of whatever size you wish
2. EXT4 to fill up the rest of the space

The first partition is the boot partition. It should be at least 200M but I suggest going higher. In fact I just broke my Raspberry Pi because the boot partition wasn't large enough for the latest kernel update.

```
fdisk /dev/<hard drive name>
o to clear all partitions

n to create a new partition
p for primary
ENTER to accept default partition number, should be 1
ENTER to accept the default first sector
+300M to specify 300 megabytes for the last sector
t for type, then c to specify FAT32 (LBA)

n then p to create a second priamry partition
ENTER to accept default partition number, should be 2
ENTER, ENTER to accept both default sectors

w to write the partition table and exit fdisk
```

Make sure to replace `<hard drive name>` with the name of your hard drive, e.g. `sdb`.

> From now on I will be assuming the hard drive is called `sdb` but be sure to replace it if that is not the name of your hard drive.

Now you'll need to format the filesystems of the partitions you created.

```
mkfs.vfat -F 32 /dev/sdb1
mkfs.ext4 /dev/sdb2
```

## Downloading and Extracting the OS

### Required for everyone

Head over to [the Arch Linux ARM downloads page](https://archlinuxarm.org/about/downloads) and decide which version you need. I will be using `http://os.archlinuxarm.org/os/ArchLinuxARM-rpi-aarch64-latest.tar.gz` for this example. This is the version you would need to install if you planned to use it on a Raspberry Pi 3/4/5.

Download the tarball.

```
wget http://os.archlinuxarm.org/os/ArchLinuxARM-rpi-aarch64-latest.tar.gz
```

Now you'll need to mount the partitions you created earlier.

```
mkdir boot
mkdir root
mount /dev/sdb1 boot
mount /dev/sdb2 root
```

Next, unpack that OS tarball into `root`, and move all of the boot files into `boot` like so:

```
bsdtar -xpf ArchLinuxARM-rpi-aarch64-latest.tar.gz -C root
mv root/boot/* boot
rm -r root/boot
```

Make sure to replace `ArchLinuxARM-rpi-aarch64-latest.tar.gz` if that is not the version you downloaded.

### Required only for Raspberry Pi 4 (and maybe 5, but I don't have one to know)

You'll need to edit the `boot.txt` file in the `boot` directory.

```
nano boot/boot.txt
```

Change <u>**all instances**</u> of `fdt_addr_r` to simply `fdt_addr`.

Then quit nano with Ctrl+X.

As it said in the first comment of the file, "After modifying, run `./mkscr`".

```
./mkscr
```

### Finally

Finally, sync with the hard drive and unmount the two partitions

```
sync
umount boot root
```

## Concluding Remarks and Useful Links

If you ran into any problems, please contact me using the footer links below.

I would also love it if you have anything to add or you just want to give some feedback.

- Raspberry Pi 4 install video [[↗]](https://www.youtube.com/watch?v=0DMxIe7l6yY)
- Olde install guide [[↗]](https://elinux.org/ArchLinux_Install_Guide)
- Arch Linux setup guide [[↗]](https://www.youtube.com/watch?v=68z11VAYMS8&t=564s)
- Arch Linux maintenance guide [[↗]](https://wiki.archlinux.org/title/system_maintenance)
- Boot failure? [[↗]](https://archlinuxarm.org/forum/viewtopic.php?f=67&t=15422&start=10#p67207)
- Firewall setup guide [[↗]](https://wiki.archlinux.org/title/simple_stateful_firewall)
