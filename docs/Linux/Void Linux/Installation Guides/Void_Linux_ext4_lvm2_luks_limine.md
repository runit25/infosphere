---
title: "Void Linux Install Guide (LUKS2 + LVM)"
description: "Installation guide for Void Linux with full disk encryption (LUKS2 + LVM), limine bootloader, and ext4."
date: 2025-12-02
tags: []
parent: Void Linux Guides
---

# Void Linux Install Guide

<!-- Created by https://gitlab.com/runit25/infosphere -->

Installation includes full disk encryption (LUKS2 + LVM), limine bootloader, ext4, and base system.

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
|2      | 1130496        | 976773134    | 475.9G | 8309 | Linux Filesystem |
```

### 7.0 Encrypt Root Partition (LUKS2): 
```shell
cryptsetup luksFormat --type luks2 --cipher aes-xts-plain64 --iter-time 5000 --key-size 256 --pbkdf argon2id /dev/nvme0n1p2
```
You’ll be prompted to enter a passphrase.

### 8.0 Open Encrypted Volume 
```shell
cryptsetup luksOpen /dev/nvme0n1p2 lukspart
```
This exposes the decrypted volume at /dev/mapper/lukspart.

### 9.0 Create LVM Physical Volume & Volume Group
```shell
pvcreate /dev/mapper/lukspart
vgcreate vg0 /dev/mapper/lukspart
```

### 10.0 Create Logical Volumes
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

### 11.0 Format Filesystems
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

### 12.0 Mount Filesystems
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
xbps-install -Sy -R https://repo-de.voidlinux.org/current/ -r /mnt base-system linux linux-firmware dracut e2fsprogs lvm2 cryptsetup xtools-minimal dhcpcd iwd openssh nano bash
```
Remove openssh if you don't intend to use it.

## Configure the System
### 1.0 Generate fstab
```shell
xgenfstab /mnt > /mnt/etc/fstab
```
xtools-minimal provides xgenfstab, which is used to generate a functional fstab. 

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

#### Fallback DNS (optional but helpful)
```shell
echo "nameserver 1.1.1.1" > /etc/resolv.conf
```
dhcpcd will overwrite this on reboot unless made immutable (chattr +i), so only use temporarily.

### 6.0 Enable Crypt, LVM in the Initramfs
```shell
mkdir -p /etc/dracut.conf.d
echo 'add_dracutmodules+=" crypt lvm "' > /etc/dracut.conf.d/90-crypt-lvm.conf
```

### 7.0 Microcode (CPU-specific)
#### AMD
```shell
xbps-install -S linux-firmware-amd
```

#### Intel
```shell
xbps-install -S intel-ucode
```

### 8.0 graphics Drivers
#### AMD
```shell
xbps-install -S mesa-dri mesa-vulkan-radeon
```
### Intel
```shell
xbps-install -S mesa-dri xf86-video-intel mesa-vulkan-intel intel-media-driver
```

### 9.0 Enable multilib and nonfree
```shell
xbps-install -S void-repo-multilib void-repo-nonfree
xbps-install -Su
```
Allows you to install Steam, etc.

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
echo 'permit persist yourusername' > /etc/doas.conf
chmod 600 /etc/doas.conf
```

### 13.0 Install limine
```shell
xbps-install -S limine
```

### 14.0 limine Bootloader
```shell
mkdir -p /boot/EFI/BOOT
cp /usr/share/limine/BOOTX64.EFI /boot/EFI/BOOT/
```

### 15.0 Get LUKS Partition UUID 
```shell
# Exit chroot
exit

# Retrieve and remember the UUID for step 16.0
blkid -s UUID -o value /dev/nvme0n1p2

# Navigate back to chroot
chroot /mnt /bin/bash
```

### 16.0 Create `/boot/limine.conf`
```shell
/Void
PROTOCOL: linux
KERNEL_PATH: boot():/vmlinuz-*
CMDLINE: rd.luks.uuid=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx root=/dev/vg0/root rw rootfstype=ext4 add_efi_memmap vsyscall=none quiet loglevel=3
MODULE_PATH: boot():/initramfs-*.img
```

Replace xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx with the actual UUID obtained from Step 15.0

replace `*` with the appropriate package versions located here `ls /boot` (e,g.,initramfs-6.12.59_1.img, vmlinuz-6.12.59_1).

`quiet loglevel=3` flag reduces boot noise (remove if debugging)

Note: Package versions are expected to be maintained in /boot/limine.conf

#### Regenerate initramfs
```shell
xbps-reconfigure -fa
```

### 17.0 Fix /boot Permissions
```shell
chmod 755 /boot
chmod 644 /boot/limine.conf
```

### 18.0 Harden /tmp, /var and /home mount options
Edit `/etc/fstab` to include:
```shell
/dev/vg0/home    /home ext4    rw,relatime,nodev    0 2

/dev/vg0/var    /var  ext4    rw,relatime,nodev,noexec,nosuid    0 2

tmpfs    /tmp    tmpfs    rw,relatime,noatime,nosuid,nodev    0 2
```
nano/vim works

### 19.0 Auto-Clean /tmp on Boot
```shell
mkdir -p /etc/tmpfiles.d
echo 'D /tmp 1777 root root 1d' > /etc/tmpfiles.d/clean-tmp.conf
```

### 20.0 Finalize and Reboot
#### Exit chroot:
```shell
exit
```

#### Unmount all partitions:
```shell
umount -R /mnt
```

#### Reboot into the new system:
```shell
reboot
```

### Post-Install
#### 1.0 Installation Checks
```shell
lsblk
swapon --show
cat /proc/cmdline
cat /boot/limine.cfg
```

### 2.0 Enable Networking Services
#### Wired
```shell
ln -s /etc/sv/dhcpcd /var/service/
```

#### WI-FI (iwd)
```shell
ln -s /etc/sv/dhcpcd /var/service/
ln -s /etc/sv/iwd /var/service/
```

## Security and Privacy enhancements
### 1.0 Randomize MAC Address
Consult: [Void_Linux_Mac_Randomization](https://gitlab.com/runit25/infosphere/-/blob/main/docs/Linux/Void%20Linux/Void%20Linux%20Enhancements/Void_Linux_Mac_Randomization.md)

### 2.0 DNS + Filtering (Recommended)
incomplete