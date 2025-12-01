---
title: "Void Linux Install Guide (LVM)"
description: "Installation guide for Arch Linux with LVM, limine bootloader, and ext4."
date: 2025-11-26
tags: []
parent: Void Linux
---

# Void Linux Install Guide

<!-- Created by https://gitlab.com/runit25/infosphere -->

Installation includes LVM, limine bootloader, ext4, and base system.

**Note:** Substitute /dev/nvme0nX with your corresponding drive (**Example:** "/dev/sda")

## Pre-installation
### 1.0 Switch to bash
```shell
/bin/bash
```

### 2.0 Verify UEFI Boot Mode
#### Ensure you're in UEFI mode:
```shell
ls /sys/firmware/efi/efivars
```
If the directory exists you're free to continue.

### 3.0 Set Keyboard Layout
```shell
loadkeys uk
```
Adjust keymap as needed (e.g., us, de). 

### 4.0 Connect to the Internet
#### Wired (DHCP):
```shell
dhcpcd
```

#### Wi-Fi (using wpa_supplicant)
```shell
wpa_supplicant -B -i wlan0 -c <(wpa_passphrase "SSID" "password")

dhcpcd wlan0
```

#### Test connectivity:
```shell
ping -c 3 voidlinux.org
```

### 5.0 List Disks
```shell
lsblk
```
Identify your target disk (e.g., /dev/nvme0n1)

### 6.0 Partition the Disk
```shell
cfdisk /dev/nvme0n1
```

```shell
# delete existing partition to make room for your new partition scheme
select [ Delete ] 

# Set up boot partition
select [ New ] 

Partition Size: 1G

select [ Type ] "EFI System"
# Set up root partition
select [ New ] 

Partition Size: accept default value

select [ Write ]
# cfdisk output
|Number | Start (sector) | End (sector) | Size   | Code | Name             |
|------ | -------------- | ------------ | ------ | ---- | ---------------- |
|1      | 2048           | 1130495      | 1G     | EF00 | EFI System       |
|2      | 1130496        | 976773134    | 475.9G | 8309 | Linux Filesystem |
```

### 7.0 Create LVM Physical Volume & Volume Group
```shell
pvcreate /dev/nvme0n1p2
vgcreate vg /dev/nvme0n1p2
```

### 8.0 Create Logical Volumes
#### Create dedicated logical volumes for better isolation and security:
```shell
# Root: OS and packages (20G)
lvcreate -L 20G vg -n root

# /var: logs, caches, databases (20G)
lvcreate -L 20G vg -n var

# /tmp: temporary files (8G)
lvcreate -L 8G vg -n tmp

# Swap: 4G (adjust to match RAM if hibernating)
lvcreate -L 4G vg -n swap

# /home: remaining space
lvcreate -l 100%FREE vg -n home
```
Adjust lvm volumes accordingly. (**Example:** "256G drive Reduce /var to 10G, /tmp to 4G")

### 9.0 Format Filesystems
```shell
# Format EFI partition
mkfs.vfat -F32 /dev/nvme0n1p1

# Format the remaining directories:
mkfs.ext4 -L "Void Root" /dev/vg/root
mkfs.ext4 -L "Void Var"  /dev/vg/var
mkfs.ext4 -L "Void Tmp"  /dev/vg/tmp
mkfs.ext4 -L "Void Home" /dev/vg/home

# Format swap
mkswap /dev/vg/swap      # Format swap LV
```

### 10.0 Mount Filesystems
```shell
# Mount root first
mount /dev/vg/root /mnt

# Create the directory structure:
mkdir -p /mnt/{boot,home,var,tmp}

# Mount EFI parition:
mount /dev/nvme0n1p1 /mnt/boot

# Mount the remaining directories:
mount /dev/vg/home /mnt/home
mount /dev/vg/var  /mnt/var
mount /dev/vg/tmp  /mnt/tmp

# Enable Swap
swapon /dev/vg/swap
```
/boot must remain unencrypted for UEFI boot. 

## Void Base Installation
#### Install Essential Packages:
```shell
xbps-install -Sy -R https://repo-de.voidlinux.org/ -r /mnt linux linux-firmware dracut e2fsprogs lvm2 dhcpcd iwd openssh nano bash
```
openssh (optional) remove unless you use ssh

## Configure the System
### 1.0 Generate fstab
```shell
xgenfstab /mnt > /mnt/etc/fstab
```

### 2.0 Chroot into the new system
```shell
chroot /mnt /bin/bash
```

### 3.0 Enable the LVM service
```shell
ln -s /etc/sv/lvm /var/service/
```

### 4.0 Set Time and Locale
```shell
ln -sf /usr/share/zoneinfo/UTC /etc/localtime # Avoids DST issues
echo "en_GB.UTF-8 UTF-8" >> /etc/default/libc-locales
xbps-reconfigure -f glibc-locales
echo "LANG=en_GB.UTF-8" > /etc/locale.conf
```

### 5.0 Network Configuration
#### Set hostname:
```shell
echo myhostname > /etc/hostname
```
#### Edit /etc/hosts:
```shell
127.0.0.1   localhost
::1         localhost
127.0.1.1   myhostname.localdomain   myhostname
```

### 6.0 Enable Networking Services
```shell
ln -s /etc/sv/dhcpcd /var/service/        # for wired
xbps-install -S iwd                       # if using Wi-Fi
ln -s /etc/sv/iwd /var/service/
```

### 7.0 Install Microcode
#### AMD
```shell
xbps-install -S linux-firmware-amd
```

#### Intel
```shell
xbps-install -S intel-ucode
```

### 8.0 Install graphics Drivers
#### AMD
```shell
xbps-install -S mesa-dri vulkan-radeon
```
### Intel
```shell
xbps-install -S mesa-dri xf86-video-intel vulkan-intel intel-media-driver
```

### 9.0 Enable multilib
echo 'multilib=yes' >> /etc/xbps.d/00-multilib.conf
xbps-install -S void-repo-multilib

### 10.0 Set Root Password
```shell
passwd
```

### 11.0 Add User
```shell
useradd -m -G wheel,users,audio,video,storage -s /bin/bash yourusername
passwd yourusername
```

### 12.0 Install opendoas
```shell
xbps-install -S opendoas
```

#### Allow yourusername to run commands as root:
```shell
echo 'permit persist :wheel' > /etc/doas.conf
chmod 600 /etc/doas.conf
```

### 13.0 Install limine
```shell
pacman -S limine
```

### 14.0 Install limine Bootloader
```shell
mkdir -p /boot/EFI/BOOT
cp /usr/local/share/limine/BOOTX64.EFI /boot/EFI/BOOT/
```

### 15.0 Create `/boot/limine.conf`
``` shell
TIMEOUT=10

:Void Linux
PROTOCOL=linux
KERNEL_PATH=boot:///vmlinuz-*
CMDLINE=root=/dev/vg0/root rw rootfstype=ext4 add_efi_memmap vsyscall=none
MODULE_PATH=

:Void Linux (Fallback)
PROTOCOL=linux
KERNEL_PATH=boot:///vmlinuz-*
CMDLINE=root=/dev/vg0/root rw rootfstype=ext4 add_efi_memmap vsyscall=none earlyprintk=efi
MODULE_PATH=
```

### 16.0 Fix /boot Permissions
```shell
chmod 755 /boot
chmod 600 /boot/limine.conf
```

## Security Hardening
### Find /tmp and include noatime, nosuid, nodev:
```shell
UUID=example    /tmp    ext4    rw,relatime,noatime,nosuid,nodev    0 2
```

### Auto-Clean /tmp on Boot
```shell
echo 'D /tmp 1777 root root 1d' > /etc/tmpfiles.d/clean-tmp.conf
```

### Privacy: Randomize MAC Address
incomplete

### DNS + Filtering (Recommended)
incomplete

## Finalize and Reboot
### Exit chroot:
```shell
exit
```

### Unmount all partitions:
```shell
umount -R /mnt
```

### Reboot into the new system:
```shell
reboot
```

### Post-Install Verification
#### Login
```shell
lsblk
swapon --show
cat /proc/cmdline
cat /boot/limine.cfg
```