# Arch Linux install

<!-- Created by https://gitlab.com/runit25/infosphere -->

Installation includes secure boot, limine bootloader, ext4, and luks2+base install.

**Note:** substitute `/dev/nvme0nX` with your corresponding drive (**Example:** "/dev/sda")

## Pre-installation
### Verify boot mode
#### If you're using UEFI, `/sys/firmware/efi/efivars` should exist
```shell
# If the directory doesn't exist your using Legacy BIOS mode
ls /sys/firmware/efi/efivars
```

### Set locale keyboard
```shell
localectl set-keymap uk
```

### Connect to the internet with Wi-Fi
```shell
iwctl
device list # list the devices
station <wlan> scan # scan for networks
station <wlan> get-networks # list the networks
station <wlan> connect <SSID> # connect to the network

# test connection
ping archlinux.org
```

#### Display your disks and partitions
```shell
lsblk
```

```shell
# example output:
|NAME           | MAJ:MIN | RM | SIZE   | RO  | TYPE  | MOUNTPOINT |
| ------------- | ------- | -- | ------ | --- | ----- | ---------- |
|nvme0n1        |  259:0  | 0  | 476.9G |  0  | disk  |            |
|├─nvme0n1p1    |  259:4  | 0  |        |  0  | part  |            |
|├─nvme0n1p2    |  259:5  | 0  |        |  0  | part  |            |
```

### Partition the disk
```shell
cfdisk /dev/nvme0n1
# delete existing partition to make room for your new partition scheme
select [ Delete ] 

# Set up boot partition
select [ New ] 

Parition Size: 1G

select [ Type ] "EFI System"
# Set up root partition
select [ New ] 

Parition Size: accept default value

select [ Write ]
# cfdisk output
|Number | Start (sector) | End (sector) | Size   | Code | Name             |
|------ | -------------- | ------------ | ------ | ---- | ---------------- |
|1      | 2048           | 1130495      | 1G     | EF00 | EFI System       |
|2      | 1130496        | 976773134    | 475.9G | 8309 | Linux Filesystem |
```

### Encrypt the root partition (luks2)
```shell
cryptsetup luksFormat \
  --type luks2 \
  --cipher aes-xts-plain64 \
  --key-size 512 \
  --hash sha512 \
  --pbkdf argon2id \
  /dev/nvme0n1p2

# Confirm and set strong passphrase
```
### Open the encrypted partition
```shell
cryptsetup luksOpen /dev/nvme0n1p2 lukspart
```

### Mount the root partition
```shell
# Format the (encrypted) ext4
mkfs.ext4 -L "Arch Linux" /dev/mapper/lukspart
# Mount filesystem
mount /dev/mapper/lukspart /mnt
```

### Prepare the EFI partition
```shell
mkfs.fat -F32 /dev/nvme0n1p1
mkdir -p /mnt/boot
mount /dev/nvme0n1p1 /mnt/boot
```

## Arch Base Installation
### Install necessary packages
(openssh) is optional, but be sure to install it if you're going to use (SSH)
```shell
pacstrap -K /mnt base base-devel linux linux-firmware polkit git mkinitcpio bash-completion dhcpcd iwd openssh nano
```

## Configure your system
### Generate the fstab file
```shell
genfstab -U /mnt >> /mnt/etc/fstab
```

### Chroot into the system
```shell
arch-chroot /mnt
```

#### By this point you should have the following partitions and logical volumes:
```shell
lsblk
```

```shell
# Expected output
| NAME         | MAJ:MIN | RM | SIZE   | RO | TYPE  | MOUNTPOINT            |
| ------------ | ------- | -- | ------ | -- | ----- | --------------------- |
| nvme0n1      | 259:0   | 0  | 475.9G | 0  | disk  |                       |
| ├─nvme0n1p1  | 259:5   | 0  | 1G     | 0  | part  | /boot                 |
| ├─nvme0n1p2  | 259:6   | 0  | 464.9G | 0  | part  |                       |
| ..└─lukspart | 254:0   | 0  | 464.9G | 0  | crypt | /.snapshots           |
|              |         |    |        |    |       | /var/lib/docker       |
|              |         |    |        |    |       | /var/cache/pacman/pkg |
|              |         |    |        |    |       | /var/log              |
|              |         |    |        |    |       | /tmp                  |
|              |         |    |        |    |       | /home                 |
|              |         |    |        |    |       | /                     |
```

### Configuring Locales
#### Secure Time and Locale
```shell
timedatectl set-ntp true
timedatectl set-timezone UTC  # More secure; avoids DST issues
hwclock --systohc --utc
```

#### Uncomment `en_GB.UTF-8 UTF-8` in:
```shell
nano /etc/locale.gen
```

```diff
-#en_GB.UTF-8 UTF-8
+en_GB.UTF-8 UTF-8
```

#### Regenerate locale file
```shell
locale-gen
```

#### set locale language, time and keyboard
```shell
localectl set-locale LANG="en_GB.UTF-8"
localectl set-locale LC_TIME="en_GB.UTF-8"
localectl set-keymap uk
```

### Network configuration
#### Create a hostname file
```shell
echo myhostname > /etc/hostname
```

Note: This is a unique name used to identify your machine on a network

#### Add matching entries to hosts
```shell
nano /etc/hosts
```

```shell
127.0.0.1 localhost
::1       localhost
127.0.1.1 myhostname
```

### Set up 4G swap file
```shell
# Create the directory for swap file
mkdir /var/swap
chmod 755 /var/swap

# Create swap file (4G)
dd if=/dev/zero of=/var/swap/swapfile bs=1M count=4096 status=progress
chmod 600 /var/swap/swapfile

# Format as swap
mkswap /var/swap/swapfile
swapon /var/swap/swapfile

# Add to fstab
echo '/var/swap/swapfile none swap defaults,noauto 0 0' >> /etc/fstab
```

### Initramfs
#### Add `encrypt` hook to `/etc/mkinitcpio.conf`
Note: ordering matters
```shell
HOOKS=(base udev autodetect microcode modconf kms keyboard keymap consolefont block encrypt filesystems fsck)
```

### Recreate the initramfs image
```shell
mkinitcpio -P
```

### Enable networking services
```shell
systemctl enable dhcpcd
systemctl enable iwd.service
systemctl enable sshd # Optionally enable if you're going to use SSH
```

#### Install microcode for the latest CPU features and security updates
```shell
# AMD CPU:
pacman -S amd-ucode 
# Intel CPU:
pacman -S intel-ucode

mkinitcpio -p linux
```

#### Graphical Drivers
##### Intell/AMD Cards
```shell
pacman -S mesa
```

##### Nvidia Cards
```shell
pacman -S nvidia nvidia-utils

sudo nano /etc/mkinitcpio.conf
# edit Modules and Files
MODULES=(nvidia nvidia_modeset nvidia_uvm nvidia_drm ...)
FILES="/etc/modprobe.d/nvidia.conf"

sudo mkinitcpio -P
nano /etc/modprobe.d/nvidia.conf
# Add row to the file
options nvidia_drm modeset=1
```

#### Enable multilib
```shell
# Uncomment the `[multilib]` section in `/etc/pacman.conf`:
[multilib]
Include = /etc/pacman.d/mirrorlist

# refresh your package databases
pacman -Syu
```

### Set your Root password
```shell 
passwd
```

### Add Linux User
```shell
useradd -m -G wheel,storage,power -s /bin/bash yourusername
passwd yourusername
```

### Opendoas (doas) allows you to run commands as root
```shell
pacman -S opendoas

# Allow <yourusername> to execute root commands
echo "permit persist keepenv yourusername" > /etc/doas.conf

# Restrict `/etc/doas.conf` permissions
chmod 600 /etc/doas.conf
```

#### (Optional) Set up a sudo alias for opendoas; Recommended if you're going to use (.sh scripts)
```shell
echo "alias sudo=doas" >> /home/yourusername/.bashrc
```

### Bootloader
```shell
# Install limine
pacman -S limine dosfstools mtools
```

#### Copy the bootloader files
```shell
mkdir -p /boot/EFI/BOOT
cp /usr/share/limine/BOOTX64.EFI /boot/EFI/BOOT/
```

#### Retrieve 'PARTUUID' from the luks partition
```shell
blkid /dev/nvme0n1p2
# Output: /dev/nvme0n1p2: PARTUUID="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
```

```shell
nano /boot/limine.conf
```

```shell
# Designates 5 second timer until Limine automatically boots
timeout: 5

/Arch Linux (linux)
    protocol: linux
    path: boot():/vmlinuz-linux
    # replace with intel-ucode.img if the target has an Intel CPU.
    module_path: boot():/amd-ucode.img
    module_path: boot():/initramfs-linux.img
    # replace "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" with your "/dev/nvme0n1p2" PARTUUID
    cmdline: cryptdevice=PARTUUID=your-partuuid-here:lukspart root=/dev/mapper/lukspart rw rootfstype=ext4

/Arch Linux (linux-fallback)
    protocol: linux
    path: boot():/vmlinuz-linux
    # replace with intel-ucode.img if the target has an Intel CPU.
    module_path: boot():/amd-ucode.img
    module_path: boot():/initramfs-linux-fallback.img
    # replace "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" with your "/dev/nvme0n1p2" PARTUUID
    cmdline: cryptdevice=PARTUUID=your-partuuid-here:lukspart root=/dev/mapper/lukspart rw rootfstype=ext4
```

#### Restrict `/boot` permissions
```shell
chmod 700 /boot
chmod 600 /boot/limine.conf
```

Congratulations the installation is now complete.
```shell
# Exit chroot and reboot
exit
umount -R /mnt
reboot
```