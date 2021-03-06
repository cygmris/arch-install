# Michael's Arch Linux installation guide

Arch Linux does not have an automated installation process. Instead, it offers a live CD that boots to a prompt with a minimal set of packages and scripts that can be used to install a system and an accompanying installation guide, available at https://wiki.archlinux.org/index.php/installation_guide.

The guide is comprehensive but takes a long time to grok in its entirety. Hence I have written this guide, that documents the specific steps I take to install Arch Linux on my ThinkPad x220, Dell xps13 and Zotec Nano AD10. It is also a learning exercise in Arch Linux and some 'low level' Linux topics.

This guide was originally written in July 2017 and last updated in November 2017. If you're reading this guide much later, then things are likely to have moved on and you may want to consult the Arch Wiki for more up to date techniques and components, etc.

It's worth remembering that, although the installation is a fairly manual process, the tasks we are completing in this installation are not very different from other operating system installations. In brief, we are:

* Partitioning (and in this instance, encrypting) the disk(s)
* Choosing regional settings (e.g. keyboard layout and time-zone)
* Downloading and installing packages
* Configuring a boot loader

## Installation media

Download an installation image and and create the installation media.

A recent installation image can be downloaded from  https://www.archlinux.org/download/.

You can create a bootable USB drive as follows:

1. insert the USB drive and note the device name
2. run `dd if=archlinux.img of=/dev/sdX bs=16M && sync` where X is the device letter

See https://wiki.archlinux.org/index.php/USB_flash_installation_media for more details.

## UEFI setup

This depends on your UEFI firmware.

### ThinkPad x220

Press `F1` after switching on the computer to enter `ThinkPad Setup`. You can reset most settings to their defaults by choosing `Load Setup Defaults` in the `Restart` menu.

In the `Startup` > `Boot` screen, ensure that you can boot from the USB HDD and also from the primary hard-drive (so that when you reboot and remove the USB drive, you boot from the hard-drive).

For extra security, you may wish to password protect `ThinkPad Setup` with a Supervisor Password (in Secuity > Password) and remove all devices from the boot priority order apart from your primary hard-drive.

## Booting the installation media

Ensure that you are booting in UEFI mode and that you can boot from USB. Insert the installion media into the ThinkPad and boot. Hit F12 while the machine is booting to bring up a menu that will allow you to boot from the USB drive. Select the USB drive to boot the install image.

The ThinkPad should boot up and log you in as the root user.

If you are booting a to an HD screen, you probably want to set the resolution. You can do this by typing e when the boot menu appears and appending `video=1366×768` to the end of the string (https://wiki.archlinux.org/index.php/kernel_parameters#systemd-boot).

## Pre-installation

There are a few things to do before installing the base system.

### Set the keyboard

Set up the keyboard so that the keyboard works as expected with `loadkeys uk`. or something similar for different keyboards. See https://wiki.archlinux.org/index.php/Keyboard_configuration_in_console for more information.

### Connect to the internet

Connect to the internet so you can download packages as part of this install.

The Arch installer uses `netctl` ('a CLI-based tool used to configure and manage network connections via profiles'). Any plugged in ethernet connection should connect automatically. The simplest way to connect to a connect to a wireless network is by using `wifi-menu`.

Alternatively, you can create a new connection profile manually based on one of the examples in /etc/netctl/examples. For more information, try `man netctl` or the [netctl](https://wiki.archlinux.org/index.php/Netctl) page on the Arch Linux wiki.

At this point, it probably makes sense to test network connectivity with a ping to google.com or similar.

### Update the system clock

Once you are connected to the internet, synchronise to network time with the following command: `timedatectl set-ntp true`

## Partition the disks

I have a 128GB SSD and want to keep things simple with as few partitions as possible. One partition would be ideal but an encrypted UEFI system will require at least two.

Create two partitions as follows:

* a large encrypted partition mounted at / (the capacity of the machine - 500M)
* a 500M unencrypted partition for UEFI booting at /boot

Use gdisk to create a GPT partition table and three partitions as follows:

| Partition            | Size  | Hex code | Device name |
|----------------------|-------|----------|-------------|
| Encrypted root       | -500M | 8300     | /dev/sda1   |
| EFI system partition | +500M | EF00     | /dev/sda2   |

Start gdisk with `gdisk /dev/sda` and delete the existing partition table with `o`.

Add the new partitions specified in the table above with the `n` command (once for each partition).

Write the table to disk with `w` and exit.

Note that the values in the *Size* and *Hex code* columns are written as they should be passed to gdisk (size should be passed as *Last sector*). Also note: passing a negative value for  *Last sector* will ensures that amount of space is left on the disk after the partition.

### Encrypt the root partition

Use `cryptsetup` to create an encrypted disk on /dev/sda1

`cryptsetup -y -v luksFormat /dev/sda1`

Open the encrypted drive so it is available at `/dev/mapper/cryptroot` with `cryptsetup open /dev/sda1 cryptroot`.

For more information on encryption, see https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#Simple_partition_layout_with_LUKS.

### Format the drives

Format /dev/mapper/cryptroot and as ext4.

`mkfs.ext4 /dev/mapper/cryptroot`

Format /dev/sd2 as FAT32.

`mkfs.fat -F32 /dev/sda2`

### Mount the partitions

Mount the partitions as follows (we need to mount /mnt before mounting /mnt/boot as we need to create the mount point before mounting).

`mount /dev/mapper/cryptroot /mnt`
`mkdir /mnt/boot`
`mount /dev/sda2 /mnt/boot`

## Installation

The pacstrap script is designed for the initial installation of packages to a disk. It also takes care of creating directories like /etc (and more stuff as well - see https://git.archlinux.org/arch-install-scripts.git/tree/pacstrap.in or type `pacstrap` with no parameters for more information.

Run `pacstrap /mnt base` to create an initial base system.

Create an fstab file based on the above mounts as follows:

`genfstab -U /mnt >> /mnt/etc/fstab`

chroot into /mnt to continue the installation with the installers special chroot script `arch-chroot`

`arch-chroot /mnt`

Do some basic configuration of locales and timezones, etc.

Note that a couple of the following commands (those to do with the keyboard and time) repeat what we did in 'Pre-installation'. We repeat them here so that they persist to the drives we mounted and hence the installed system.

`ln -sf /usr/share/zoneinfo/Europe/London /etc/localtime`
`hwclock --systohc`
`echo "LANG=en_GB.UTF-8" > /etc/locale.conf`
`echo "KEYMAP=uk" > /etc/vconsole.conf`
`echo "thinkpad" > /etc/hostname`
`echo "127.0.1.1	thinkpad.localdomain	thinkpad" >> /etc/hosts`

Uncomment `en_US.UTF-8 UTF-8` and `en_GB.UTF-8 UTF-8` in `/etc/locale.gen`

Generate the locales with `locale-gen`.

### Microcode updates

We need to install microcode updates. See https://wiki.archlinux.org/index.php/microcode for more information.

`pacman -S intel-ucode`

### Wireless networking

Installing `pacman -S iw wpa_supplicant dialog` makes it simple to connect wirelessly after booting up the installed system. Unless you want to fiddle around with a lot of command It's probably worth installing at this point in time in any case, if you need to connect wirelessly.

TODO: ensure that I need to carry out this step

## Build a custom initramfs

We need to build an initramfs that will work for us. To do this we edit `/etc/mkinitcpio.conf` and run `mkinitcpio -p linux` when finished.

Read the `/etc/mkinitcpio.conf` file and https://wiki.archlinux.org/index.php/Mkinitcpio for more information.

## Ensuring the USB keyboard is available early on

Add the `keyboard` hook.

## Unencrypting the encrypted partition

Add the `encrypt` hook. Note that hooks are executed in order. Hence if you want to use your USB keyboard to enter the passphrase, and have the prompt displayed in a nice font, ensure that the `encrypt` hook appears after `keyboard` and `consolefont`.

## Setting a nice console font

Ensure you have access to the console font you want to use (`pacman -S terminus-font`).

Add a font line to /etc/vconsole.conf, for example `FONT=ter-u32n`.

Add the `consolefont` hook to `/etc/mkinitcpio.conf`.

## Boot loader

Configure the UEFI bootloader.

Install the systemd-bootloader with `bootctl --path=/boot install`. This copies the systemd-boot binary to the EFI System Partition here: /boot/EFI/systemd/systemd-bootx64.efi and also here: /boot/EFI/Boot/BOOTX64.EFI (both files are identical). It also adds systemd-boot to the boot loader and sets it as the default.

We now need to configure the bootloader.

Open `/boot/loader/loader.conf` and ensure that it looks like this

```
default arch
timeout 0
editor 0
```

This sets `/boot/efi/loader/entries/arch.conf` as the default entry. Since we only have one entry it makes sense to set timeout to 0 which will boot the default immediately.

Create a new file `/boot/efi/loader/entries/arch.conf` to contain details of the Arch Linux system we want to boot. The file should look similar to the following example.

```
title Arch Linux
linux /vmlinuz-linux
initrd /intel-ucode.img
initrd /initramfs-linux.img
options cryptdevice=UUID=284e9b4f-07ed-41eb-8838-84ec79f42f4a:cryptroot root=/dev/mapper/cryptroot
```

* `title` is required. It would show in the boot menu if the timeout wasn't set to 0.
* `linux` points to the linux kernal (relative to the boot partition root).
* `initrd` points to the initial ramdisk (relative to the boot partition root). It is also used to enable Intel microcode updates
* `options` is used to pass options to the kernel

The options are kernel options that are passed to the kernel. They merit a little more explanation. We need to add a `cryptdevice` option to give bootloader details of the encrypted drive. The bootloader will then prompt for the password of the encrypted drive at startup. `root` tells the kernal which device to mount as the root filesystem (the name is derived from the device mapper specified in cryptdevice: /dev/mapper/cryptroot).

Note: the UUID passed to cryptdevice is the UUID of the **parent** device that contains the encrypted device, not the UUID of the encrypted device itself. i.e in this case, it is the UUID of `/dev/sda1`, not of `/dev/mapper/cryptroot`.

You can find the UUID of the encrypted device with `lsblk -f` as long as it has been opened.

If you are interested in how UEFI works, then this is a great resource: https://www.happyassassin.net/2014/01/25/uefi-boot-how-does-that-actually-work-then/.


### Reboot

Exit the chroot environment by typing `exit`.

Reboot the system by typing `reboot`

## Post installation configuration

### Configure user accounts

Configure a password for root with `passwd`.

Create a user for michael `useradd -m -G wheel michael`.

Let users in the wheel group do passwordless sudo by commenting out the appropriate line in  `/etc/sudoers`.

## Useful packages

Install the following useful packages and package groups with `pacman -S...`

```
atom
chromium
firefox
git
gnome
iw
sudo
vim
wpa_supplicant
zsh
zsh-completions
```

See https://wiki.archlinux.org/index.php/General_recommendations for a recommended list of things to do once Arch is installed.
and https://wiki.archlinux.org/index.php/List_of_applications for some ideas on packages that you should install

# Desktop environment

Install the gnome desktop environment and display manager with `pacman -S gnome`.

Enable the gdm login at boot by enabling the systemd service `systemctl enable gdm.service`

# TODO

* Work out if I should switch from netctl to systemd-networkd
