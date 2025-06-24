# Arch Linux install

<!-- Created by https://gitlab.com/runit25/infosphere -->

Installation includes Secure Boot, BTRFS+Backup, LUKS+Base Install

**Note:** substitute `/dev/nvme0nX` with your corrosponding drive (**Example:** "/dev/sda")

## Pre-installation
### Verify boot mode
if you're using UEFI, `/sys/firmware/efi/efivars` should exist
```sh
# If the directory doesn't exist your using Legacy BIOS mode
ls /sys/firmware/efi/efivars
```

### Connect to the internet
**Ethernet consult** 
```shell
Plug in your ethernet cable and go
```

**Wireless consult**
```shell
iwctl
device list # list the devices
station <wlan> scan # scan for networks
station <wlan> get-networks # list the networks
station <wlan> connect <SSID> # connect to the network

# test connection
ping archlinux.org
```

### Update your system clock
```shell
timedatectl set-ntp true
```

#### Display your disks and partitions
```shell
lsblk

# example output:
|NAME           | MAJ:MIN | RM | SIZE   | RO  | TYPE  | MOUNTPOINT |
| ------------- | ------- | -- | ------ | --- | ----- | ---------- |
|nvme0n1        |  259:0  | 0  | 476.9G |  0  | disk  |            |
|├─nvme0n1p1    |  259:4  | 0  |        |  0  | part  |            |
|├─nvme0n1p2    |  259:5  | 0  |        |  0  | part  |            |
```

### Partition your disk
```shell
cfdisk /dev/nvme0n1
# delete existing partition to make room for your new partition scheme
select [ Delete ] 

# Setup boot partition
select [ New ] 

Parition Size: 1G

select [ Type ] "EFI System"
# setup root partition
select [ New ] 

Parition Size: accept default value

select [ Write ]
# cfdisk output
|Number | Start (sector) | End (sector) | Size   | Code | Name             |
|------ | -------------- | ------------ | ------ | ---- | ---------------- |
|1      | 2048           | 1130495      | 1G     | EF00 | EFI System       |
|2      | 1130496        | 976773134    | 475.9G | 8309 | Linux Filesystem |
```

#### Create LUKS2 encrypted container on your Linux Filesystem as seen above

```shell
cryptsetup luksFormat --use-urandom -S 1 -s 512 -h sha512 -i 5000 /dev/nvme0n1p2
```

#### Open the container (decrypt it to make available at /dev/mapper/lukspart)
```shell
cryptsetup luksOpen /dev/nvme0n1p2 lukspart
```

### Prepare BTRFS Volume/Subvolumes
#### Formatting to btrfs now that it's already encrypted
```shell
mkfs.btrfs -L "Arch Linux" /dev/mapper/lukspart
```

#### Mount filesystem
```shell
mount /dev/mapper/lukspart /mnt
```

#### Create subvolumes

This scheme can be adjusted to your needs, I'd suggest at least one subvolume for root (@) and one for snapshots (.@snapshots). varlog and tmp are created to disable copy on Write on /var/log and /tmp.

```shell
btrfs su cr /mnt/@
btrfs su cr /mnt/@home
btrfs su cr /mnt/@tmp
btrfs su cr /mnt/@log
btrfs su cr /mnt/@pkg
btrfs su cr /mnt/@docker
btrfs su cr /mnt/.@snapshots
```

#### Disable copy on write on /var/log and /tmp

```shell
chattr +C /mnt/@log
chattr +C /mnt/@tmp  
umount /mnt
```

#### Mount BTRFS Subvolumes

```shell
mount -o defaults,noatime,discard,ssd,subvol=@ /dev/mapper/lukspart /mnt  
mkdir -p /mnt/{home,var/log,var/cache/pacman/pkg,var/lib/docker,tmp,.snapshots}

# Discard and ssd options are exclusive for ssd disks only
mount -o defaults,noatime,discard,ssd,subvol=@home /dev/mapper/lukspart /mnt/home
mount -o defaults,noatime,discard,ssd,subvol=@tmp /dev/mapper/lukspart /mnt/tmp
mount -o defaults,noatime,discard,ssd,subvol=@log /dev/mapper/lukspart /mnt/var/log
mount -o defaults,noatime,discard,ssd,subvol=@pkg /dev/mapper/lukspart /mnt/var/cache/pacman/pkg/
mount -o defaults,noatime,discard,ssd,subvol=@docker /dev/mapper/lukspart /mnt/var/lib/docker
mount -o defaults,noatime,discard,ssd,subvol=.@snapshots /dev/mapper/lukspart /mnt/.snapshots
```

### Preparing your EFI partition
#### Create FAT32 filesystem on the EFI system partition
```shell
mkfs.fat -F32 /dev/nvme0n1p1
```

#### Create mountpoint for EFI system partition at /efi for compatibility with grub-install and mount it
```shell
mkdir /mnt/efi
mount /dev/nvme0n1p1 /mnt/efi
```

## Arch Base Installation
### Install necessary packages
(openssh) is optional, but be sure to install it if you're going to use (SSH)
```shell
pacstrap -K /mnt/ base base-devel linux linux-firmware polkit git btrfs-progs efibootmgr mkinitcpio bash-completion dhcpcd iwd openssh opendoas nano
```

#### Enable multilib (required by certain packages such as `steam`)
Uncomment the `[multilib]` section in `/etc/pacman.conf`:
```shell
[multilib]
Include = /etc/pacman.d/mirrorlist
```

## Configure your system
### Generate the fstab file
```shell
genfstab -U /mnt >> /mnt/etc/fstab
```

### Enter new system chroot
```shell
arch-chroot /mnt
```

#### TODO: (Replace it) At this point you should have the following partitions and logical volumes:
```shell
lsblk

# Expected output
| NAME         | MAJ:MIN | RM | SIZE   | RO | TYPE  | MOUNTPOINT            |
| ------------ | ------- | -- | ------ | -- | ----- | --------------------- |
| nvme0n1      | 259:0   | 0  | 475.9G | 0  | disk  |                       |
| ├─nvme0n1p1  | 259:5   | 0  | 1G     | 0  | part  | /efi                  |
| ├─nvme0n1p2  | 259:6   | 0  | 464.9G | 0  | part  |                       |
| ..└─lukspart | 254:0   | 0  | 464.9G | 0  | crypt | /.snapshots           |
|              |         |    |        |    |       | /var/lib/docker       |
|              |         |    |        |    |       | /var/cache/pacman/pkg |
|              |         |    |        |    |       | /var/log              |
|              |         |    |        |    |       | /tmp                  |
|              |         |    |        |    |       | /home                 |
|              |         |    |        |    |       | /                     |
```

### Time zone
#### Set your time zone
Replace `Europe/London` with your respective timezone found in `/usr/share/zoneinfo`
```shell
ln -sf /usr/share/zoneinfo/Europe/London /etc/localtime
```

#### Run `hwclock` to generate ```/etc/adjtime```
Assumes hardware clock is set to UTC
```shell
hwclock --systohc --utc
```

### Localization
#### Uncomment ```en_GB.UTF-8 UTF-8``` in ```/etc/locale.gen``` and generate locale
```shell
locale-gen
```

#### Create ```locale.conf``` and set your ```LANG``` variable
```shell
echo LANG=en_GB.UTF-8 > /etc/locale.conf
export LANG=en_GB.UTF-8
```

### Network configuration
#### Create the hostname file
```shell
echo myhostname > /etc/hostname
```

This is a unique name for identifying your machine on a network.

#### Add matching entries to hosts
```shell
nano /etc/hosts
```
```shell
127.0.0.1 localhost
::1       localhost
127.0.1.1 myhostname
```

### Initramfs
#### Add ```encrypt```, and ```btrfs``` hooks to ```/etc/mkinitcpio.conf```
*Note:* ordering matters.
```shell
HOOKS=(base udev autodetect modconf kms keyboard keymap consolefont block encrypt btrfs filesystems fsck)

# Add btrfsck to binaries
BINARIES=(btrfsck)
```

#### Recreate the initramfs image
```shell
mkinitcpio -P
```

### Enabled Networking services
```shell
systemctl enable dhcpcd
systemctl enable iwd.service
systemctl enable sshd # Optionally enable if you're going to use SSH
```

### Install microcode for the latest CPU features and security updates
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

### Root password
#### Set the root password
```shell
passwd
```

### Add Linux User
```shell
useradd -m -g users -G wheel,storage,power -s /bin/bash <user>
passwd <user>
pacman -S opendoas
EDITOR=nano /etc/doas.conf
# Allow <user> to execute root commands
permit persist keepenv <user>
```

### (Optional) Setup a sudo alias for opendoas; Recommended if you're going to use (.sh scripts)
```shell
nano .bashrc
# sudo alias for opendoas
alias sudo='doas'
```

### Boot loader
#### Install GRUB
```shell
pacman -S grub efibootmgr os-prober dosfstools mtools
```

#### Configure GRUB to allow booting from /boot on a LUKS2 encrypted partition
```shell
nano /etc/default/grub
```
```
GRUB_ENABLE_CRYPTODISK=y
```

#### Uncomment if you're duel-booting
```shell
GRUB_DISABLE_OS_PROBER=false
```

#### Set kernel parameter to unlock the BTRFS physical volume at boot using ```encrypt``` hook
##### UUID is the partition containing the LUKS container
```shell
blkid
```
```shell
/dev/nvme0n1p2: UUID="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" TYPE="crypto_LUKS" PARTLABEL="Linux LUKS" PARTUUID="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
```
```shell
nano /etc/default/grub
```
```shell
# allow-discards is only for ssd to let trim work with encryption enabled
GRUB_CMDLINE_LINUX="... cryptdevice=UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx:lukspart:allow-discards"
```

#### Install GRUB to the mounted ESP for UEFI booting
```shell
grub-install --target=x86_64-efi --bootloader-id=ARCH --efi-directory=/efi --recheck
```

#### Generate GRUB's configuration file
```
grub-mkconfig -o /boot/grub/grub.cfg
```

### (recommended) Embed a keyfile in initramfs

This is done to avoid having to enter your encryption passphrase twice (once for GRUB, once for initramfs.)

#### Create a keyfile and add it to the LUKS key
```shell
mkdir /root/secrets && chmod 700 /root/secrets
head -c 64 /dev/urandom > /root/secrets/crypto_keyfile.bin && chmod 600 /root/secrets/crypto_keyfile.bin
cryptsetup -v luksAddKey -i 1 /dev/nvme0n1p2 /root/secrets/crypto_keyfile.bin
```

#### Add the keyfile to the initramfs image
```shell
nano /etc/mkinitcpio.conf
```
```shell
FILES=(/root/secrets/crypto_keyfile.bin)
```

#### Recreate the initramfs image
```shell
mkinitcpio -P
```

#### Set kernel parameters to unlock the LUKS partition with the keyfile using ```encrypt``` hook
```shell
/etc/default/grub
```
```shell
GRUB_CMDLINE_LINUX="... cryptkey=rootfs:/root/secrets/crypto_keyfile.bin"
```

#### Regenerate GRUB's configuration file
```shell
grub-mkconfig -o /boot/grub/grub.cfg
```

#### Restrict ```/boot``` permissions
```shell
chmod 700 /boot
```

The installation is now complete. Exit the chroot and reboot.
```shell
exit
reboot
```