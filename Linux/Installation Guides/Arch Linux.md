# Arch Linux install

<!-- Created by https://gitlab.com/runit25/infosphere -->

Installation includes Secure Boot, Limine Bootloader, BTRFS+Backup, LUKS+Base Install

**Note:** substitute `/dev/nvme0nX` with your corrosponding drive (**Example:** "/dev/sda")

## Pre-installation
### Verify boot mode
#### if you're using UEFI, `/sys/firmware/efi/efivars` should exist
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

### Encrypt the partition
```shell
# Encrypt the root partition
cryptsetup luksFormat --use-urandom -S 1 -s 512 -h sha512 -i 5000 /dev/nvme0n1p2

# Open the encrypted partition
cryptsetup luksOpen /dev/nvme0n1p2 lukspart
```

#### Prepare the BTRFS subvolumes
```shell
# Format (encrypted) btrfs
mkfs.btrfs -L "Arch Linux" /dev/mapper/lukspart
# Mount filesystem
mount /dev/mapper/lukspart /mnt
```

#### Configure the BTRFS subvolumes
```shell
btrfs su cr /mnt/@
btrfs su cr /mnt/@home
btrfs su cr /mnt/@tmp
btrfs su cr /mnt/@log
btrfs su cr /mnt/@pkg
btrfs su cr /mnt/@docker
btrfs su cr /mnt/.@snapshots

# Disable copy on write (CoW) on "/var/log" and "/tmp"
chattr +C /mnt/@log
chattr +C /mnt/@tmp  
umount /mnt
```

#### Mount BTRFS Subvolumes
```shell
mount -o defaults,noatime,ssd,subvol=@ /dev/mapper/lukspart /mnt  
mkdir -p /mnt/{home,var/log,var/cache/pacman/pkg,var/lib/docker,tmp,.snapshots}

# SSD mount option is exlusive to SSD disks only
mount -o defaults,noatime,ssd,subvol=@home /dev/mapper/lukspart /mnt/home
mount -o defaults,noatime,ssd,subvol=@tmp /dev/mapper/lukspart /mnt/tmp
mount -o defaults,noatime,ssd,subvol=@log /dev/mapper/lukspart /mnt/var/log
mount -o defaults,noatime,ssd,subvol=@pkg /dev/mapper/lukspart /mnt/var/cache/pacman/pkg/
mount -o defaults,noatime,ssd,subvol=@docker /dev/mapper/lukspart /mnt/var/lib/docker
mount -o defaults,noatime,ssd,subvol=.@snapshots /dev/mapper/lukspart /mnt/.snapshots
```

### Prepare the EFI partition
```shell
# Create a FAT32 filesystem on the EFI system partition
mkfs.fat -F32 /dev/nvme0n1p1

# Create a mount point for the EFI system partition at /boot for compatibility with Limine
mkdir /mnt/boot
mount /dev/nvme0n1p1 /mnt/boot
```

## Arch Base Installation
### Install necessary packages
(openssh) is optional, but be sure to install it if you're going to use (SSH)
```shell
pacstrap -K /mnt/ base base-devel linux linux-firmware polkit git btrfs-progs mkinitcpio bash-completion dhcpcd iwd openssh nano
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
#### Enable Time Sync
```shell
timedatectl set-ntp true
```

#### Set your timezone
```shell
# Check available timezones
timedatectl list-timezones

# Example location output (Europe/London)
timedatectl set-timezone Europe/London
```

#### Sync hardware clock
```shell
hwclock --systohc
```

#### Uncomment `en_GB.UTF-8 UTF-8` in `/etc/locale.gen` and generate locale
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
# Create swap subvolume
btrfs subvolume create /var/swap
# Create sparse file (efficient)
truncate -s 0 /var/swap/swapfile
chattr +C /var/swap/swapfile
chmod 600 /var/swap/swapfile
fallocate -l 4G /var/swap/swapfile
mkswap --label swapfile /var/swap/swapfile
swapon /var/swap/swapfile
echo '/var/swap/swapfile none swap defaults,x-systemd.device-timeout=2,noauto 0 0' >> /etc/fstab
```

### Initramfs
#### Add `encrypt`, and `btrfs` hooks to `/etc/mkinitcpio.conf`
Note: ordering matters
```shell
HOOKS=(base udev autodetect microcode modconf kms keyboard keymap consolefont block encrypt btrfs filesystems fsck)

# Add btrfsck to binaries
BINARIES=(btrfsck)
```

### Recreate the initramfs image
```shell
mkinitcpio -P
```

### Enabled Networking services
```shell
systemctl enable dhcpcd
systemctl enable iwd.service
systemctl enable sshd # Optionally enable if you're going to use SSH
```

#### Enable multilib
```shell
# Uncomment the `[multilib]` section in `/etc/pacman.conf`:
[multilib]
Include = /etc/pacman.d/mirrorlist

# refresh your package databases
pacman -Syu
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

### Set your Root password
```shell 
passwd
```

### Add Linux User
```shell
useradd -m -G wheel,storage,power -s /bin/bash <user>
passwd <user>
```

### opendoas (doas) allows you to run root Commands
```shell
pacman -S opendoas

nano /etc/doas.conf
# Allow <user> to execute root commands
permit persist keepenv <user>
```

#### (Optional) Set up a sudo alias for opendoas; Recommended if you're going to use (.sh scripts)
```shell
nano ~/.bashrc
# sudo alias for opendoas
alias sudo="doas"
```
```shell
# Restrict `/etc/doas.conf` permissions
chmod 600 /etc/doas.conf
```

### Bootloader
```shell
# Install limine
pacman -S limine dosfstools mtools
```

#### Copy Bootloader files
```shell
mkdir -p /boot/EFI/BOOT/
cp -v /usr/share/limine/BOOTX64.EFI /boot/EFI/BOOT/
ls /usr/share/limine
```

#### Set the kernel parameter to unlock the BTRFS physical volume at boot 
UUID is the partition containing the LUKS container
```shell
blkid
```

```shell
/dev/nvme0n1p2: UUID="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" TYPE="crypto_LUKS" PARTLABEL="Linux LUKS" PARTUUID="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
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
    cmdline: cryptdevice=PARTUUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx:root root=/dev/mapper/root rootflags=subvol=@ rw rootfstype=btrfs

/Arch Linux (linux-fallback)
    protocol: linux
    path: boot():/vmlinuz-linux
    # replace with intel-ucode.img if the target has an Intel CPU.
    module_path: boot():/amd-ucode.img
    module_path: boot():/initramfs-linux-fallback.img
    # replace "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" with your "/dev/nvme0n1p2" PARTUUID
    cmdline: cryptdevice=PARTUUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx:root root=/dev/mapper/root rootflags=subvol=@ rw rootfstype=btrfs
```

#### Restrict `/boot` permissions
```shell
chmod 700 /boot
```

Congratulations the installation is now complete.
```shell
# Exit choort and reboot
exit
reboot
```