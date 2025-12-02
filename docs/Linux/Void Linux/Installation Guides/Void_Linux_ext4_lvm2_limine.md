---
title: "Void Linux Install Guide (LVM)"
description: "Installation guide for Void Linux with LVM, Limine bootloader, and ext4."
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
Adjust accordingly (e.g., us, de).

### 4.0 Connect to the Internet
#### Wired (DHCP):
```shell
dhcpcd
```

#### Wi-Fi (using wpa_supplicant)
```shell
wpa_passphrase "SSID" "password" > /tmp/wpa.conf
wpa_supplicant -B -i wlan0 -c /tmp/wpa.conf
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
|2      | 1130496        | 976773134    | 475.9G | 8300 | Linux Filesystem |
```

### 7.0 Create LVM Physical Volume & Volume Group
```shell
pvcreate /dev/nvme0n1p2
vgcreate vg0 /dev/nvme0n1p2
```

### 8.0 Create Logical Volumes
#### Create dedicated logical volumes for better isolation and security:
```shell
# Root: OS and packages (20G)
lvcreate -L 20G vg0 -n root

# /var: logs, caches, databases (20G)
lvcreate -L 20G vg0 -n var

# /tmp: temporary files (8G)
lvcreate -L 8G vg0 -n tmp

# Swap: 4G (adjust to match RAM if hibernating)
lvcreate -L 4G vg0 -n swap

# /home: remaining space
lvcreate -l 100%FREE vg0 -n home
```
Drives (<128G), reduce /var to 10G, /tmp to 4G.

### 9.0 Format Filesystems
```shell
# Format EFI partition
mkfs.vfat -F32 /dev/nvme0n1p1

# Format the remaining directories:
mkfs.ext4 -L "Void_Root" /dev/vg0/root
mkfs.ext4 -L "Void_Var"  /dev/vg0/var
mkfs.ext4 -L "Void_Tmp"  /dev/vg0/tmp
mkfs.ext4 -L "Void_Home" /dev/vg0/home

# Format swap
mkswap /dev/vg0/swap    # Format swap LV
```

### 10.0 Mount Filesystems
```shell
# Mount root first
mount /dev/vg0/root /mnt

# Create the directory structure:
mkdir -p /mnt/{boot,home,var,tmp}

# Mount EFI partition:
mount /dev/nvme0n1p1 /mnt/boot

# Mount the remaining directories:
mount /dev/vg0/home /mnt/home
mount /dev/vg0/var  /mnt/var
mount /dev/vg0/tmp  /mnt/tmp

# Enable Swap
swapon /dev/vg0/swap
```
/boot must remain unencrypted for UEFI boot. 

## Void Base Installation
#### Install Essential Packages:
```shell
xbps-install -Sy -R https://repo-de.voidlinux.org/current/ -r /mnt base-system linux linux-firmware dracut e2fsprogs lvm2 xtools-minimal dhcpcd openssh nano bash
```
Remove openssh if you don't intend to use it.

## Configure the System
### 1.0 Generate fstab
```shell
xgenfstab /mnt > /mnt/etc/fstab
```

### 2.0 Chroot into the new system
```shell
chroot /mnt /bin/bash
```

### 3.0 Set Time and Locale
```shell
ln -sf /usr/share/zoneinfo/Region/City /etc/localtime  # e.g., Europe/London
echo "en_GB.UTF-8 UTF-8" >> /etc/default/libc-locales
xbps-reconfigure -f glibc-locales
echo "LANG=en_GB.UTF-8" > /etc/locale.conf
```

### 4.0 Network Configuration
#### Set hostname:
```shell
echo myhostname > /etc/hostname
```
#### Edit /etc/hosts:
```shell
cat > /etc/hosts <<EOF
127.0.0.1   localhost
::1         localhost
127.0.1.1   myhostname.localdomain myhostname
EOF
```
Alternatively use nano/vim

### 5.0 Enable Networking Services
#### Wired
```shell
ln -s /etc/sv/dhcpcd /var/service/
```

#### WI-FI (iwd)
```shell
xbps-install -S iwd
ln -s /etc/sv/iwd /var/service/
```

#### Fallback DNS (optional but helpful)
```shell
echo "nameserver 1.1.1.1" > /etc/resolv.conf
```
dhcpcd will overwrite this on reboot unless made immutable (chattr +i), so only use temporarily.

### 6.0 Microcode (CPU-specific)
#### AMD
```shell
xbps-install -S linux-firmware-amd
```

#### Intel
```shell
xbps-install -S intel-ucode
```

### 7.0 graphics Drivers
#### AMD
```shell
xbps-install -S mesa-dri mesa-vulkan-radeon
```
### Intel
```shell
xbps-install -S mesa-dri xf86-video-intel mesa-vulkan-intel intel-media-driver
```

### 8.0 Enable multilib and nonfree
```shell
xbps-install -S void-repo-multilib void-repo-nonfree
xbps-install -Su
```
Allows you to install Steam, etc.

### 9.0 Set Root Password
```shell
passwd
```

### 10.0 Add User
```shell
useradd -m -G wheel,users,audio,video,storage -s /bin/bash yourusername
passwd yourusername
```

### 11.0 Install opendoas
```shell
xbps-install -S opendoas
```

#### Allow yourusername to run commands as root:
```shell
echo 'permit persist yourusername' > /etc/doas.conf
chmod 600 /etc/doas.conf
```

### 12.0 Install limine
```shell
xbps-install -S limine
```

### 13.0 limine Bootloader
```shell
mkdir -p /boot/EFI/BOOT
cp /usr/share/limine/BOOTX64.EFI /boot/EFI/BOOT/
```

### 14.0 Create `/boot/limine.conf`
```shell
/Void
PROTOCOL: linux
KERNEL_PATH: boot():/vmlinuz-*
CMDLINE: root=/dev/vg0/root rw rootfstype=ext4 add_efi_memmap vsyscall=none quiet loglevel=3
MODULE_PATH: boot():/initramfs-*.img
```
replace `*` with the package version located in `ls /boot` (e.g., vmlinuz-6.12.59_1). 

`quiet loglevel=3` flag reduces boot noise (remove if debugging)

#### Regenerate initramfs
```shell
xbps-reconfigure -fa
```

### 15.0 Fix /boot Permissions
```shell
chmod 755 /boot
chmod 644 /boot/limine.conf
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