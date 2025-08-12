# Void Linux Install

<!-- Created by https://gitlab.com/runit25/infosphere -->

Installation includes...... to be continued

**Note:** Substitute `/dev/nvme0nX` with your corresponding drive (**Example:** "/dev/sda")

## Pre-installation
### 1.0 Verify UEFI Boot Mode
#### Ensure you're in UEFI mode:
```shell
ls /sys/firmware/efi/efivars
```
If the directory exists you're free to continue

### 2.0 Set Keyboard Layout
```shell
loadkeys uk
```

### 3.0 Connect to the Internet

Wired
```shell

```
Wi-Fi
```shell

```

### 4.0 List Disks
```shell
lsblk
```
Identify your target disk (e.g., `/dev/nvme0n1`)

### 5.0 Partition the Disk
```shell
cfdisk /dev/nvme0n1
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

### 6.0 Create LVM Physical Volume & Volume Group
```shell
pvcreate /dev/nvme0n1p2
vgcreate vg /dev/nvme0n1p2
```

### 7.0 Create Logical Volumes
#### create dedicated LVs for better isolation and security:
```shell
# Root: OS and packages (20G)
lvcreate -L 20G  vg -n root

# /var: logs, caches, databases (20G)
lvcreate -L 20G  vg -n var

# /tmp: temporary files (8G)
lvcreate -L 8G   vg -n tmp

# Swap: 4G (adjust to match RAM if hibernating)
lvcreate -L 4G   vg -n swap

# /home: remaining space
lvcreate -l 100%FREE vg -n home
```
Adjust sizes based on total disk:

`256G` Reduce `/var` to `10G`, `/tmp` to `4G`

### 8.0 Format Filesystems
```shell
mkfs.ext4 -L "Void Root" /dev/vg/root
mkfs.ext4 -L "Void Var"  /dev/vg/var
mkfs.ext4 -L "Void Tmp"  /dev/vg/tmp
mkfs.ext4 -L "Void Home" /dev/vg/home
mkswap /dev/vg/swap
```

### 9.0 Mount Filesystems
```shell
# Mount root first
mount /dev/vg/root /mnt

# Create and mount other directories
mkdir -p /mnt/{home,var,tmp,boot}
mount /dev/vg/home /mnt/home
mount /dev/vg/var  /mnt/var
mount /dev/vg/tmp  /mnt/tmp

# Prepare and mount EFI partition
mkfs.fat -F32 /dev/nvme0n1p1
mount /dev/nvme0n1p1 /mnt/boot
```
`/boot` is unencrypted (required for UEFI boot)

## Void Base Installation
#### Install Essential Packages
```shell
xbps-install -Sy -R "https://repo-de.voidlinux.org/current"-r /mnt base-system
```
base-system includes kernel, glibc, runit, xbps, and basic tools. 

## Configure the System
### 1.0 Generate fstab
```shell

```

### 2.0 Chroot into New System
```shell

```

#### `lsblk` should look similar to this:
```shell
lsblk
```

```shell

```

### 3.0 Set Time and Locale
```shell
ln -sf /usr/share/zoneinfo/UTC /etc/localtime
hwclock --systohc --utc
```

#### Generate locales:
```shell
echo "en_GB.UTF-8 UTF-8" >> /etc/default/libc-locales
xbps-reconfigure -f glibc-locales
```

#### Set keymap persistent via `/etc/vconsole.conf`:
```shell
echo "KEYMAP=uk" > /etc/vconsole.conf
```

### 4.0 Network Configuration
#### Set hostname:
```shell
echo myhostname > /etc/hostname
```

#### Edit `/etc/hosts`:
```shell
nano /etc/hosts

# Include:
127.0.0.1   localhost
::1         localhost
127.0.1.1   myhostname.localdomain   myhostname
```

### 5.0 Enable Swap
```shell
swapon /dev/vg/swap
```

### 6.0 Initramfs Configuration (for LVM)
```shell
xbps-install -y dracut
```

#### Enable LVM module:
```shell
echo 'add_dracutmodules+=" lvm "' >> /etc/dracut.conf.d/lvm.conf
```

#### Generate initramfs:
```shell
dracut --force --hostonly
```
`--hostonly` includes only necessary modules.

### 7.0 Enable Networking Services (runit)
#### Void uses runit — enable services by creating symlinks:
```shell
ln -s /etc/sv/dhcpcd /var/service/
ln -s /etc/sv/iwd     /var/service/
ln -s /etc/sv/sshd    /var/service/ # Optional
```

### 8.0 Install Microcode
#### For AMD:
```shell
xbps-install -y amd-ucode
```

#### For Intel:
```shell
xbps-install -y intel-ucode
```

#### Regenerate initramfs to include microcode:
```shell
dracut --force --hostonly
```

### 9.0 Install Graphics Drivers
```shell
xbps-install -y mesa-dri
```

#### For Vulkan:
```shell
xbps-install -y mesa-vulkan-radeon  # AMD
xbps-install -y mesa-vulkan-intel   # Intel
```

#### Enable multilib (32-bit support):
```shell
xbps-install -Sy void-repo-multilib
xbps-install -S lib32-mesa-dri
```

### 10.0 Set Root Password
```shell
passwd
```

### 11.0 Add User
```shell
useradd -m -G wheel,users,storage,power -s /bin/bash yourusername
passwd yourusername
```

### 12.0 Install `doas`:
```shell
xbps-install -y opendoas
```

#### Configure:
```shell
echo "permit persist keepenv yourusername" > /etc/doas.conf
chmod 600 /etc/doas.conf
```

#### Optional: Add `sudo` alias:
```shell
echo "alias sudo=doas" >> /home/yourusername/.profile
```

### 13.0 Install `limine`

### Install `limine` Bootloader
```shell
mkdir -p /boot/EFI/BOOT
cp limine-4.0.1-binary/limine-efi-x86_64.bin /boot/
cp limine-4.0.1-binary/BOOTX64.EFI /boot/EFI/BOOT/
```

### Create `/boot/limine.cfg`
```shell
nano /boot/limine.cfg
```

```conf
timeout: 5

entry "Void Linux"
    protocol: linux
    kernel: boot:/vmlinuz-void
    initrd: boot:/amd-ucode.img   # Remove if Intel
    initrd: boot:/intel-ucode.img # Remove if AMD
    initrd: boot:/initramfs-void.img
    cmdline: root=/dev/vg/root rw rootfstype=ext4 add_efi_memmap

entry "Void Linux (fallback)"
    protocol: linux
    kernel: boot:/vmlinuz-void
    initrd: boot:/amd-ucode.img   # Remove if Intel
    initrd: boot:/intel-ucode.img # Remove if AMD
    initrd: boot:/initramfs-void-fallback.img
    cmdline: root=/dev/vg/root rw rootfstype=ext4 add_efi_memmap
```
Replace `vmlinuz-void` and `initramfs-void.img` with actual kernel names in `/boot`

#### List kernel:
```shell
ls /boot/vmlinuz-* /boot/initramfs-*.img
```

### 15.0 Fix `/boot` Permissions
```shell
chmod 755 /boot
chmod 600 /boot/limine.cfg
```

### 16.0 Secure `/tmp` Mount Options
#### Edit `/etc/fstab`:
```shell

```

### 17.0 (Optional) Clear /tmp on Boot
#### Void uses tmpfiles — create:
```shell
nano /etc/tmpfiles.d/clean-tmp.conf

# Include:
D /tmp 1777 root root 1d
```
Clears files older than 1 day

## Finalize and Reboot
#### Exit chroot:

```shell
exit
```

#### Unmount all:
```shell
umount -R /mnt
```

#### Reboot:
```shell
reboot
```

## Verify Installation
#### After logging in:
```shell
lsblk
swapon --show
cat /boot/limine.cfg
cat /etc/fstab
doas whoami
```

## You Now Have:

✅ Void Linux base system with runit

✅ LVM with separate /, /home, /var, /tmp

✅ limine bootloader (UEFI)

✅ Secure /tmp with noexec,nosuid,nodev

✅ UTC time, proper locales, networking

✅ opendoas, user account, microcode

✅ Minimal, secure, and maintainable setup