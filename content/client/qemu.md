---
## Guide Information
title: "QEMU"
description: "Run high-performance virtual machines."
icon: qemu.svg 
date: 2024-03-06T14:40:00-05:00
#ports: [80, 443]

## Author Information
author: Denshi
#email:
#xmpp:
#matrix:
#monero:

draft: true
---

QEMU is emulation software.

## Installation

Most Linux distributions provide a package for QEMU. An updated list of installation methods is present [here.](https://www.qemu.org/download/#linux)

## Creating a Disk

```sh
qemu-img create -f qcow2 {{<hl>}}image.img{{</hl>}} {{<hl>}}10G{{</hl>}}
```

## Launching the VM

```sh
qemu-system-x86_64 -enable-kvm -cdrom {{<hl>}}diskimage.iso{{</hl>}} -boot menu=on -drive file={{<hl>}}image.img{{</hl>}} -m {{<hl>}}2G{{</hl>}}
```

> Note: Use <kbd>Ctrl</kbd> + <kbd>Alt</kbd> + <kbd>G</kbd> to release the grab from the QEMU window.

### Performance options

The following flags may be enabled to increase performance:

- `-cpu host` -- sets the CPU to the hosts' CPU
- `-smp {{<hl>}}2{{</hl>}}` -- Sets the numbers of cores
- `-vga virtio` -- Enables graphics acceleration
- `-device virtio-vga-gl -display gtk,gl=on` -- Enables 3D acceleration

An example launch command using some of these flags would look like this:

```sh
qemu-system-x86_64 -enable-kvm -cdrom {{<hl>}}diskimage.iso{{</hl>}} -boot menu=on -drive file={{<hl>}}image.img{{</hl>}} -m {{<hl>}}8G{{</hl>}} -cpu host -smp {{<hl>}}4{{</hl>}} -vga virtio
```

### Enabling UEFI

If you need to run a desktop virtual machine using UEFI, begin by installing OVMF.
