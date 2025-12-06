---
title: "Arch Linux Install Guide (LVM)"
description: "Installation guide for Arch Linux with LVM, limine bootloader, ext4, and security hardening."
date: 2025-12-06
tags: []
parent: Arch Linux Guides
---

# Arch Linux Install Guide

<!-- Created by https://gitlab.com/runit25/infosphere -->

Installation includes LVM, limine bootloader, ext4, base system, and security hardening.

**Note:** Substitute `/dev/nvme0nX` with your corresponding drive (**Example:** `/dev/sda`).

## Pre-installation
### 1.0 Verify UEFI Boot Mode
#### Ensure you're in UEFI mode:
```shell
ls /sys/firmware/efi/efivars
```
If the directory exists you're free to continue.

### 2.0 Set Keyboard Layout
```shell
loadkeys uk
```
Adjust accordingly (e.g., us, de).

### 3.0 Connect to the Internet
#### Wired (DHCP):
```shell
dhcpcd
```

#### Wi-Fi (using iwd)
```shell
iwctl
device list                  # Identify interface (e.g., wlan0)
station wlan0 scan           # Scan networks
station wlan0 get-networks   # List networks
station wlan0 connect 'SSID' # Replace SSID with your network name
exit
```

#### Test connectivity:
```shell
ping -c 3 archlinux.org
```

### 4.0 List Disks
```shell
lsblk
```
Identify your target disk (e.g., /dev/nvme0n1).

### 5.0 Partition the Disk
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

### 6.0 Create LVM Physical Volume & Volume Group
```shell
pvcreate /dev/nvme0n1p2
vgcreate vg0 /dev/nvme0n1p2
```

### 7.0 Create Logical Volumes
#### Create dedicated logical volumes for better isolation and security:
```shell
# Root: OS and packages (20G)
lvcreate -L 20G vg0 -n root

# /var: logs, caches, databases (20G)
lvcreate -L 20G vg0 -n var

# Swap: 4G
lvcreate -L 4G vg0 -n swap

# /home: remaining space
lvcreate -l 100%FREE vg0 -n home
```
Drives (<128G), reduce /var to 10G.

Note: /tmp is handled securely via tmpfs in RAM (see hardening section).

### 8.0 Format Filesystems
```shell
# Format EFI partition
mkfs.fat -F32 /dev/nvme0n1p1

# Format the remaining directories:
mkfs.ext4 -L "Arch_Root" /dev/vg0/root
mkfs.ext4 -L "Arch_Var"  /dev/vg0/var
mkfs.ext4 -L "Arch_Home" /dev/vg0/home

# Format swap
mkswap /dev/vg0/swap
```

### 9.0 Mount Filesystems
```shell
# Mount root first
mount /dev/vg0/root /mnt

# Create the directory structure:
mkdir -p /mnt/{boot,home,var}

# Mount EFI partition:
mount /dev/nvme0n1p1 /mnt/boot

# Mount the remaining directories:
mount /dev/vg0/home /mnt/home
mount /dev/vg0/var  /mnt/var

# Enable Swap
swapon /dev/vg0/swap
```
/boot must remain unencrypted for UEFI boot.

## Arch Base Installation
#### Install Essential Packages:
```shell
pacstrap /mnt base linux linux-firmware lvm2 mkinitcpio bash-completion dhcpcd iwd openssh nano
```
Remove openssh if you don't intend to use it.

## Configure the System
### 1.0 Generate fstab
```shell
genfstab -U /mnt >> /mnt/etc/fstab
```

### 2.0 Chroot into the new system
```shell
arch-chroot /mnt
```

### 3.0 Set Time and Locale
```shell
timedatectl set-ntp true
timedatectl set-timezone UTC # Avoids DST issues
hwclock --systohc --utc
```

#### Uncomment en_GB.UTF-8 UTF-8 in /etc/locale.gen (Adjust accordingly):
```shell
nano /etc/locale.gen
```
vim works as well

#### Generate and set locale (Adjust accordingly):
```shell
locale-gen
localectl set-locale LANG="en_GB.UTF-8"
localectl set-locale LC_TIME="en_GB"
echo "KEYMAP=uk" > /etc/vconsole.conf
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

### 5.0 Initramfs Configuration
#### Edit /etc/mkinitcpio.conf:
```shell
nano /etc/mkinitcpio.conf
```
vim works as well

```shell
# Include vfat
MODULES=(vfat)

# Replace systemd with udev and sd-vconsole with consolefont, and add lvm2 (ordering matters)
HOOKS=(base udev autodetect microcode modconf kms keyboard keymap consolefont block lvm2 filesystems fsck)
```

### 6.0 Microcode (CPU-specific)
#### AMD
```shell
pacman -S amd-ucode
```
#### Intel
```shell
pacman -S intel-ucode
```

#### Regenerate initramfs:
```shell
mkinitcpio -P
```

### 7.0 Graphics Drivers
#### AMD
```shell
pacman -S mesa-dri mesa-vulkan-radeon lib32-mesa
```

#### Intel
```shell
pacman -S mesa-dri xf86-video-intel mesa-vulkan-intel intel-media-driver lib32-mesa
```

### 8.0 Enable multilib
#### Edit /etc/pacman.conf:
```shell
[multilib]
Include = /etc/pacman.d/mirrorlist
```

#### Then:
```shell
pacman -Sy
```

### 9.0 Set Root Password
```shell
passwd
```

### 10.0 Add User
```shell
useradd -m -G wheel,storage,power -s /bin/bash yourusername
passwd yourusername
```

### 11.0 Install opendoas
```shell
pacman -S opendoas
```

#### Allow yourusername to run commands as root:
```shell
echo 'permit persist yourusername' > /etc/doas.conf
chmod 600 /etc/doas.conf
```

### 12.0 Install limine
```shell
pacman -S limine
```

### 13.0 limine Bootloader
```shell
mkdir -p /boot/EFI/BOOT
cp /usr/share/limine/BOOTX64.EFI /boot/EFI/BOOT/
```

### 14.0 Create /boot/limine.conf
```shell
nano /boot/limine.conf
```
vim works as well

```shell
timeout: 10

/Arch
protocol: linux
path: boot():/vmlinuz-linux
module_path: boot():/amd-ucode.img        # Delete this line if you're using intel
module_path: boot():/intel-ucode.img      # Delete this line if you're using amd
module_path: boot():/initramfs-linux.img
cmdline: root=/dev/vg0/root rw rootfstype=ext4 add_efi_memmap vsyscall=none quiet loglevel=3
```
`quiet loglevel=3` flag reduces boot noise (remove if debugging)

vim works as well

### 15.0 Fix /boot Permissions
```shell
chmod 755 /boot
chmod 644 /boot/limine.conf
```

### 16.0 Harden /tmp, /var, and /home mount options
Edit /etc/fstab:
```shell
nano /etc/fstab
```
vim works as well

```shell
# /home: no device access
/dev/vg0/home    /home  ext4  rw,relatime,nodev                0 2

# /var: no exec, suid, or device access
/dev/vg0/var     /var   ext4  rw,relatime,nodev,noexec,nosuid  0 2

# /tmp: RAM-backed, ephemeral, hardened
tmpfs            /tmp   tmpfs rw,relatime,noatime,nosuid,nodev,noexec,mode=1777  0 0
```

### 17.0 Auto-Clean /tmp on Boot
```shell
mkdir -p /etc/tmpfiles.d
echo 'D /tmp 1777 root root 1d' > /etc/tmpfiles.d/clean-tmp.conf
```

### 18.0 Enable Networking Services
```shell
systemctl enable dhcpcd
systemctl enable iwd.service
systemctl enable sshd        # Ignore unless you use sshd
```

### 19.0 Finalize and Reboot
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

## Post-Install
### 1.0 Installation Checks
```shell
lsblk
swapon --show
cat /proc/cmdline
cat /boot/limine.conf
```

## Security and Privacy Enhancements
### 1.0 Randomize MAC Address
Consult: [Arch_Linux_Mac_Randomization](<https://gitlab.com/runit25/infosphere/-/blob/main/docs/Linux/Arch%20Linux/Arch%20Linux%20Enhancements/Arch_Linux_Mac_Randomization.md>)

### 2.0 DNS + Filtering
incomplete

### 3.0 Kernel Self-Protection
```shell
# Prevent kernel pointer leaks
echo 'kernel.dmesg_restrict = 1' > /etc/sysctl.d/51-dmesg-restrict.conf
echo 'kernel.kptr_restrict = 2' >> /etc/sysctl.d/51-dmesg-restrict.conf

# Harden ASLR
echo 'kernel.randomize_va_space = 2' >> /etc/sysctl.d/51-dmesg-restrict.conf
```