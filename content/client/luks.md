---
## Guide Information
title: "LUKS Encryption"
description: "Encrypt external and internal disks on Linux."
icon:  luks.svg
date: 2024-07-07T11:16:42-05:00
#ports: [80, 443]

## Author Information
author: Denshi
---

LUKS (Linux Unified Key Setup) is a libre and free disk encryption standard which allows for the encryption of external and internal disks on Linux. Everything from an SD card to your root partition can be encrypted with LUKS, allowing you to keep your data secure.

LUKS works by utilizing a **master encryption key** protected by a **user key**, which in most cases will be a password.

## External USB Drive

Suppose the device `/dev/sdb1` is a USB flash stick partition we wish to encrypt. To do so, simply run:

```sh
cryptsetup luksFormat {{<hl>}}/dev/sdb1{{</hl>}}
```

You will be prompted to enter a password and verify it. **Make sure to pick a secure one!**

### Accessing the external drive

To decrypt the partition, simply run:

```sh
cryptsetup open {{<hl>}}/dev/sdb1{{</hl>}} {{<hl>}}cryptusb{{</hl>}}
```

This will make the partition device accessible in `/dev/mapper/cryptusb`. To use it properly, it must have a formatted filesystem:

```sh
mkfs.ext4 /dev/mapper/{{<hl>}}cryptusb{{</hl>}}
```

Now it may be mounted:
```sh
mount /dev/mapper/cryptusb /mnt
```

> Note: A filesystem daemon like GVFS will automatically know how to handle encrypted drives and prompt you for a password when accessing them.

## Encrypted Root Partition

Many Linux distributions with GUI installers like [Linux Mint](https://linuxmint.com/) and [Artix](https://artixlinux.org/) offer an an encrypted root partition option. Simply enable encryption and set a password in their GUI to be prompted at boot.

The rest of this guide will follow all the steps necessary to set up an encrypted root partition on a manual install distribution like [Archlinux](https://archlinux.org) or [Gentoo](https://gentoo.org).

### Partition Setup

Begin by setting up an encrypted partition as normal:

```sh
cryptsetup luksFormat {{<hl>}}/dev/sda2{{</hl>}}
# You will now be prompted for a password...

# Make note of where you open the encrypted partition!
cryptsetup open {{<hl>}}/dev/sda2{{</hl>}} {{<hl>}}cryptroot{{</hl>}}
# You will once again be prompted for a password

# Format and mount the partition
mkfs.ext4 /dev/mapper/{{<hl>}}cryptroot{{</hl>}}
mount /dev/mapper/{{<hl>}}cryptroot{{</hl>}} /mnt
```

### Installing Linux

> When installing Linux, make sure to install the packages containing the `cryptroot` and `lvm2` commands.

You may now install Linux to the partition mounted to `/mnt` as normal. However, **the partition will not boot if you leave it like this!** Make sure to follow these steps:

### `mknitcpio` setup

> If you are using **`dracut`** or **`booster`** for your initcpio, you can ignore this step!

If you are using Archlinux or Artix, the default initcpio software included with your distribution is `mkinitcpio`. This software does not support encryption by default. This can be corrected by editing `/etc/mkinitcpio.conf` and editing the `HOOKS=(...` line to include the following:

```sh
HOOKS=(base udev autodetect modconf kms keyboard keymap consolefont block {{<hl>}}encrypt{{</hl>}} {{<hl>}}lvm2{{</hl>}} filesystems fsck)
```

After modifying the config, make sure to re-generate the initcpio:

```sh
mkinitcpio -P
```

### Bootloader setup

Your bootloader needs to be aware of your encrypted partition setup, or it won't know where to look for your root partition.

#### Obtaining UUIDs

To set up encryption, you must obtain the UUIDs for both your encrypted partition (`cryptdevice`) and your **unencrypted** partition (`root`). To see these, use the `blkid` command with some special parameters:

```sh
## Obtain the UUID for {{<hl>}}cryptdevice{{</hl>}}:
blkid /dev/sda2 -o value -s UUID
## In this example, we'll use {{<hl>}}f4f9f9f6-222a-4018-b45a-9b86544890e4{{</hl>}}

## Obain the UUID for {{<blue>}}root{{</blue>}}:
blkid /dev/mapper/cryptroot -o value -s UUID
## In this example, we'll use {{<blue>}}83276439-f9fa-4429-a2e2-91c072c02c4f{{</blue>}}
```

#### GRUB Setup

Set the following in `/etc/default/grub`:

```sh
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet cryptdevice=UUID={{<hl>}}f4f9f9f6-222a-4018-b45a-9b86544890e4{{</hl>}}:cryptroot root=UUID={{<blue>}}83276439-f9fa-4429-a2e2-91c072c02c4f{{</blue>}}"
```

#### rEFInd Setup

Set the following in `/boot/refind_linux.conf`:
```sh
"Boot with standard options"    "root=UUID={{<blue>}}83276439-f9fa-4429-a2e2-91c072c02c4f{{</blue>}} ro cryptdevice=UUID={{<hl>}}f4f9f9f6-222a-4018-b45a-9b86544890e4{{</hl>}}:cryptroot:allow-discards loglevel=3"
"Boot to single-user mode"      "root=UUID={{<blue>}}83276439-f9fa-4429-a2e2-91c072c02c4f{{</blue>}} ro cryptdevice=UUID={{<hl>}}f4f9f9f6-222a-4018-b45a-9b86544890e4{{</hl>}}:cryptroot:allow-discards quiet single"
"Boot with minimal options"     "root=UUID={{<blue>}}83276439-f9fa-4429-a2e2-91c072c02c4f{{</blue>}} ro cryptdevice=UUID={{<hl>}}f4f9f9f6-222a-4018-b45a-9b86544890e4{{</hl>}}:cryptroot:allow-discards
```

Congratulations! You should now be able to reboot and run your system with LUKS encryption!
