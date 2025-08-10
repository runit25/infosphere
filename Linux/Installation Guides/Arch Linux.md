# Arch Linux install

<!-- Created by https://gitlab.com/runit25/infosphere -->

Installation includes full disk encryption (LUKS2 + LVM), limine bootloader (Secure Boot), ext4, and base system.

**Note:** substitute `/dev/nvme0nX` with your corresponding drive (**Example:** "/dev/sda")

## Pre-installation
### Verify boot mode
#### If you're using UEFI, `/sys/firmware/efi/efivars` should exist
```shell
# If the directory doesn't exist, you're using Legacy BIOS mode and this guide won't work
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

### Create LVM Physical Volume and Volume Group
```shell
pvcreate /dev/mapper/lukspart
vgcreate vg /dev/mapper/lukspart
```

### Create Logical Volumes for `/` and `/home`
```shell
# Root: 50G (adjust as needed)
lvcreate -L 50G vg -n root

# Home: rest of the space
lvcreate -l 100%FREE vg -n home
```

### Format and Mount Filesystems
```shell
# Format root
mkfs.ext4 -L "Arch Root" /dev/vg/root

# Format home
mkfs.ext4 -L "Arch Home" /dev/vg/home

# Mount root
mount /dev/vg/root /mnt

# Create and mount home
mkdir /mnt/home
mount /dev/vg/home /mnt/home
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
| NAME          | MAJ:MIN | RM | SIZE   | RO | TYPE  | MOUNTPOINT |
| ------------- | ------- | -- | ------ | -- | ----- | ---------- |
| nvme0n1       | 259:0   | 0  | 476.9G | 0  | disk  |            |
| ├─nvme0n1p1   | 259:1   | 0  |     1G | 0  | part  | /boot      |
| └─nvme0n1p2   | 259:2   | 0  | 475.9G | 0  | part  |            |
|   └─lukspart  | 254:0   | 0  | 475.9G | 0  | crypt |            |
|     ├─vg-root | 254:1   | 0  |    50G | 0  | lvm   | /          |
|     └─vg-home | 254:2   | 0  | 425.9G | 0  | lvm   | /home      |
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
echo "KEYMAP=uk" > /etc/vconsole.conf
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
echo '/var/swap/swapfile none swap defaults,noauto,sw 0 0' >> /etc/fstab
```

### Initramfs
#### Add `encrypt` hook to `/etc/mkinitcpio.conf`
Note: ordering matters
```shell
HOOKS=(base udev autodetect microcode modconf kms keyboard keymap consolefont block encrypt lvm2 filesystems fsck)
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

### Graphical Drivers
#### Intell/AMD Cards
```shell
pacman -S mesa
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

### Install limine and sbctl
```shell
pacman -S limine sbctl
```

### Generate Secure Boot Keys
```shell
sbctl generate-keys
```

#### `sbctl generate-keys` creates: 
```shell
/var/db/sbctl/owner.key # private

/var/db/sbctl/owner.crt # public, to be enrolled
```

### Enroll Keys via MokManager
```shell
sbctl enroll-keys --mok

# This adds a key to the MOK (Machine Owner Key) list.

On next boot, the MokManager (blue screen) will appear.

You must:

1. Select "Enroll MOK"

2. Select "Continue"

3. Select "Yes"

4. Enter the password you set during generate-keys

After this, the system will trust your signed binaries.
```

### Copy limine UEFI Bootloader
```shell
mkdir -p /boot/EFI/BOOT
cp /usr/share/limine/BOOTX64.EFI /boot/EFI/BOOT/
```

### receive UUID for `/boot/limine.conf`
```shell
blkid -s UUID -o value /dev/nvme0n1p2

# Example output: "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" 
```

### Create `/boot/limine.conf`:
```shell
nano /boot/limine.conf
```

```shell
# Designates a 5 second timer until Limine automatically boots
timeout: 5

/Arch Linux (linux)
    protocol: linux
    path: boot:/vmlinuz-linux
    module_path: boot:/amd-ucode.img # Use "intel-ucode.img" for Intel
    module_path: boot:/initramfs-linux.img
    cmdline: cryptdevice=UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx:lukspart root=/dev/vg/root rw rootfstype=ext4 add_efi_memmap vsyscall=none

/Arch Linux (linux-fallback)
    protocol: linux
    path: boot:/vmlinuz-linux
    module_path: boot:/amd-ucode.img      # Use intel-ucode.img for Intel
    module_path: boot:/initramfs-linux-fallback.img
    cmdline: cryptdevice=UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx:lukspart root=/dev/vg/root rw rootfstype=ext4 add_efi_memmap vsyscall=none

```

### Fix `/boot` Permissions
```shell
chmod 755 /boot
chmod 600 /boot/limine.conf
```

### Sign Bootloader and Kernel
```shell
sbctl sign -s /boot/EFI/BOOT/BOOTX64.EFI
sbctl sign -s /boot/vmlinuz-linux
sbctl sign -s /boot/initramfs-linux.img
sbctl sign -s /boot/initramfs-linux-fallback.img
sbctl sign -s /boot/amd-ucode.img        # Use intel-ucode.img for Intel
```

Run `sbctl verify` to confirm all files are signed

### Automate Signing on Updates
```shell
nano /etc/pacman.d/hooks/100-sign-secureboot.hook
```

```shell
[Trigger]
Type = Path
Operation = Upgrade
Target = /boot/vmlinuz-linux
Target = /boot/initramfs-linux.img
Target = /boot/amd-ucode.img
Target = /boot/intel-ucode.img

[Action]
Description = Signing EFI binaries for Secure Boot
When = PostTransaction
Exec = /usr/bin/sbctl sign-all
NeedsTargets
```

#### Reboot and Enroll Key (MokManager)
```shell
# Exit chroot and reboot
exit
umount -R /mnt
reboot
```

##### On reboot: 
```shell
MokManager will appear (blue screen)

Follow prompts to enroll your key

After enrollment, disable Setup Mode
```

### Enable Secure Boot in Firmware
#### After key enrollment:
```shell
Reboot into UEFI settings

Enable:

Secure Boot = Enabled

Mode = User Mode or Custom Mode

Setup Mode = False

Save and exit
```

### Verify Secure Boot is Active
#### After booting:
```shell
mokutil --sb-state            # Should say "Secure boot enabled"
sbctl status                  # Should show "Enrolled keys", "Signed binaries"
dmesg | grep -i "secure boot" # Should confirm enabled
```

You now have a fully encrypted, securely booted Arch Linux system with:

LUKS2 + LVM

ext4 root/home

limine bootloader

Secure Boot via sbctl

Proper key enrollment

Automated signing
