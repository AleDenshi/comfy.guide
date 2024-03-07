---
## Guide Information
title: "LUKS Encryption"
description: "Encrypt external and internal disks on Linux."
icon:  gpg.svg
date: 2024-03-07T11:16:42-05:00
#ports: [80, 443]

## Author Information
author: Denshi

draft: true
---

LUKS (Linux Unified Key Setup) is a libre and free disk encryption standard which allows for the encryption of external and internal disks on Linux. Everything from an SD card to your root partition can be encrypted with LUKS, allowing you to keep your data secure.

LUKS works by utilizing a **master encryption key** protected by a **user key**, which in most cases will be a password.

## External Drive

Suppose the device `/dev/sdb1` is a USB flash stick partition we wish to encrypt. To do so, simply run:

```sh
cryptsetup luksFormat {{<hl>}}/dev/sdb1{{</hl>}}
```

You will be prompted to enter a password and verify it. **Make sure to pick a secure one!**

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

(Insert guide here)
