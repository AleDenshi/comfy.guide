---
## Software Information
title: NVIDIA on Linux
date: 2023-01-01
description: A simple guide on how to use NVIDIA on Linux.
icon: nvidia.svg

## Author Information
author: Denshi
---

NVIDIA GPUs are **notoriously hard** to setup on Linux. This comprehensive guide aims to make this relatively difficult process streamlined and easy, to maximize either performance or battery life.

> This guide is aimed **specifically** at Arch Linux systems. The steps described here may not work on other distributions, but still remain indicative of the general setup required to get NVIDIA working.

## NVIDIA Optimus (Laptops)
Optimus refers to the underlying system in laptops that allows them to manage both an Intel iGPU and an NVIDIA (dedicated) GPU. There are various ways of managing Optimus, and this section of the guide will run through various options.

By running the following command, one can check what GPU is currently rendering their screen:

```sh
glxinfo | grep "renderer"
```
If no configuration has been done, this should default to the **open-source Mesa driver. **This means your NVIDIA card is not rendering the screen at the moment;

### Prerequisites
To setup NVIDIA cards on laptops, the following packages are required:
* the NVIDIA driver
* Xorg Xrandr

## Manual setup
This section covers the process of **manually creating and editing config files** to setup your system the way you want.

### PCI Bus
These sections will require you to determine the PCI bus of your NVIDIA card. This can be accomplished by running:

```sh
lspci | grep -i nvidia | awk '{print $1}'
```
> This guide will assume your PCI bus is `PCI:1:0:0`. Please change this according to your setup.

### Only use NVIDIA
Firstly, remove any blacklists or previous NVIDIA Optimus configuration:

```sh
sudo rm -rf /etc/modprobe.d/blacklist-nvidia.conf /lib/udev/rules.d/50-remove-nvidia.rules
```
Write this to `/etc/X11/xorg.conf.d/90-nvidia.conf`:

```sh
Section "ServerLayout"
    Identifier "layout"
    Screen 0 "nvidia"
    Inactive "intel"
EndSection

Section "Device"
    Identifier "nvidia"
    Driver "nvidia"
    BusID "PCI:1:0:0"
EndSection

Section "Screen"
    Identifier "nvidia"
    Device "nvidia"
    Option "AllowEmptyInitialConfiguration"
EndSection

Section "Device"
    Identifier "intel"
    Driver "modesetting"
EndSection

Section "Screen"
    Identifier "intel"
    Device "intel"
EndSection
```

Then create `/etc/modprobe.d/nvidia.conf`:

```sh
options nvidia-drm modeset=1
```

### Xinit and Display Managers
To actually enable the use of the NVIDIA card, global Xorg configuration is often not enough. The **individual display/login mangers** (or your `~/.xinitrc` file) have to be configured to properly enable NVIDIA rendering.

If using **Xinit**, add the following to your `~/.xinitrc` file at the beginning:

```sh
xrandr --setprovideroutputsource modesetting NVIDIA-0
xrandr --auto
```

If using any display managers, change the following configuration files:

#### LightDM
Create a script, `/etc/lightdm/display_setup.sh`, and add the following:

```sh
#!/bin/sh
xrandr --setprovideroutputsource modesetting NVIDIA-0
xrandr --auto
```
Make the script executable by running:

```sh
sudo chmod +x /etc/lightdm/display_setup.sh
```

Then, edit `/etc/lightdm/lightdm.conf` to execute this script when launching LightDM:

```sh
[Seat:*]
display-setup-script=/etc/lightdm/display_setup.sh
```

#### SDDM
SDDM has a convenient Xsetup file located in `/usr/share/sddm/scripts/Xsetup` that can be modified to add any parameters you wish. Add the following to enable NVIDIA:

```sh
xrandr --setprovideroutputsource modesetting NVIDIA-0
xrandr --auto
```

#### GDM
Firstly, ensure GDM is launching Xorg sessions and not Wayland sessions by editing `/etc/gdm/custom.conf`:

```sh
WaylandEnable=false
```
Then create two new files, `/usr/share/gdm/greeter/autostart/optimus.desktop` and `/etc/xdg/autostart/optimus.desktop`, and write the following contents to them:

```sh
[Desktop Entry]
Type=Application
Name=Optimus
Exec=sh -c "xrandr --setprovideroutputsource modesetting NVIDIA-0; xrandr --auto"
NoDisplay=true
X-GNOME-Autostart-Phase=DisplayServer
```

### Only use the iGPU
Firstly, remove any custom NVIDIA Xorg configuration:

```sh
sudo rm -rf /etc/X11/xorg.conf.d/90-nvidia.conf /etc/modprobe.d/nvidia.conf
```
> Ensure any display manager or Xinit configuration for NVIDIA is also removed!

Begin by blacklisting the NVIDIA card by creating `/etc/modprobe.d/blacklist-nvidia.conf`:

```sh
blacklist nouveau
blacklist nvidia
blacklist nvidia_drm
blacklist nvidia_uvm
blacklist nvidia_modeset

alias nouveau off
alias nvidia off
alias nvidia_drm off
alias nvidia_uvm off
alias nvidia_modeset off
```

Then create `/lib/udev/rules.d/50-remove-nvidia.rules` and add the following:

```sh
= Remove NVIDIA USB xHCI Host Controller devices, if present
ACTION=="add", SUBSYSTEM=="pci", ATTR{vendor}=="0x10de", ATTR{class}=="0x0c0330", ATTR{power/control}="auto", ATTR{remove}="1"

= Remove NVIDIA USB Type-C UCSI devices, if present
ACTION=="add", SUBSYSTEM=="pci", ATTR{vendor}=="0x10de", ATTR{class}=="0x0c8000", ATTR{power/control}="auto", ATTR{remove}="1"

= Remove NVIDIA Audio devices, if present
ACTION=="add", SUBSYSTEM=="pci", ATTR{vendor}=="0x10de", ATTR{class}=="0x040300", ATTR{power/control}="auto", ATTR{remove}="1"

= Finally, remove the NVIDIA dGPU
ACTION=="add", SUBSYSTEM=="pci", ATTR{vendor}=="0x10de", ATTR{class}=="0x03[0-9]*", ATTR{power/control}="auto", ATTR{remove}="1"
```

Finally, remove any NVIDIA configuration in your display manager or your `~/.xinitrc` file, if applicable.

### Hybrid system
While your Linux system will automatically run Xorg on your iGPU by default, this doesn't prevent the NVIDIA GPU from drawing **select windows** on your system. This can be achieved through **NVIDIA PRIME,** a hybrid graphics technology.

On Arch Linux, the following package can be installed:

```sh
sudo pacman -S nvidia-prime
```
After installing this, one can run any Xorg program with `prime-run` to run it under the NVIDIA GPU.

## EnvyControl
There are multiple simple and easy-to-use scripts one can install to make managing NVIDIA Optimus a breeze. One such example is [Envycontrol](https://github.com/geminis3/envycontrol), a minimal python script which can automatically switch the GPU modes and disable/blacklist the NVIDIA GPU to save power, all on demand.

### Installation
EnvyControl can be installed from source:

```sh
git clone https://github.com/geminis3/envycontrol.git
cd envycontrol
sudo pip install .
```

Alternatively, EnvyControl is available in the [AUR](https://aur.archlinux.org/packages/envycontrol/):

```sh
paru -S envycontrol
```

### Usage
EnvyControl automatically supports and can configure the following login managers:
* GDM
* SDDM
* LightDM

If using Xinit, one can always add these lines to your `~/.xinitrc` whenever NVIDIA is in use:

```sh
xrandr --setprovideroutputsource modesetting NVIDIA-0
xrandr --auto
xrandr --dpi 96
```
To actually switch between graphics modes, simply run EnvyControl as such:

```sh
sudo envycontrol --switch nvidia
```

After this, you will be prompted to reboot, and your system will be running under your specified GPU.

Please note: **Disabling the NVIDIA GPU** using `envycontrol --switch integrated` will cause the script to be *unable to switch back to NVIDIA* unless the system is switched back to `hybrid` mode.

