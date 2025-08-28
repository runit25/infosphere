# Arch Linux Install

<!-- Created by https://gitlab.com/runit25/infosphere -->

Installation includes LVM, limine bootloader, ext4, and base system.

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
localectl set-keymap uk
```
Adjust keymap as needed (e.g., us, de). 

### 3.0 Connect to the Internet 

#### Wired (DHCP):
```shell
ping -c 3 archlinux.org
```

#### Wi-Fi (using `iwd`):

```shell
iwctl
device list                # Identify interface (e.g., wlan0)
station wlan0 scan         # Scan networks
station wlan0 get-networks # List networks
station wlan0 connect SSID # Replace SSID with your network name
exit
```

#### Test connectivity:
```shell
ping -c 3 archlinux.org
```

### 4.0 List Disks
```shell
lsblk

# Example output:
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
nvme0n1     259:0    0 476.9G  0 disk 
├─nvme0n1p1 259:1    0     1G  0 part 
└─nvme0n1p2 259:2    0 475.9G  0 part 

# We will be installing Linux on 'nvme0n1'
```

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
pvcreate /dev/mapper/lukspart
vgcreate vg /dev/mapper/lukspart
```

### 7.0 Create Logical Volumes
#### create dedicated LVs for better isolation and security:
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
Adjust sizes based on total disk:

`256G` Reduce `/var` to `10G`, `/tmp` to `4G`

### 8.0 Format Filesystems
```shell
mkfs.ext4 -L "Arch Root"   /dev/vg/root
mkfs.ext4 -L "Arch Var"    /dev/vg/var
mkfs.ext4 -L "Arch Tmp"    /dev/vg/tmp
mkfs.ext4 -L "Arch Home"   /dev/vg/home
mkswap /dev/vg/swap         # Format swap LV
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

## Arch Base Installation
#### Install Essential Packages
```shell
pacstrap -K /mnt base linux linux-firmware mkinitcpio bash-completion dhcpcd iwd openssh nano
```
`openssh` (optional) remove unless you use ssh

## Configure the System
### 1.0 Generate fstab
```shell
genfstab -U /mnt >> /mnt/etc/fstab
```

### 2.0 Chroot into New System
```shell
arch-chroot /mnt
```

#### `lsblk` should look similar to this:
```shell
lsblk
```

```shell
# output
NAME          MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINT
nvme0n1       259:0    0 476.9G  0 disk  
├─nvme0n1p1   259:1    0     1G  0 part  /boot
└─nvme0n1p2   259:2    0 475.9G  0 part  
  └─vg-root   254:1    0    20G  0 lvm   /
  └─vg-var    254:2    0    20G  0 lvm   /var
  └─vg-tmp    254:3    0     8G  0 lvm   /tmp
  └─vg-swap   254:4    0     4G  0 lvm   [SWAP]
  └─vg-home   254:5    0 423.9G  0 lvm   /home
```

### 3.0 Set Time and Locale
```shell
timedatectl set-ntp true
timedatectl set-timezone UTC # Avoids DST issues
hwclock --systohc --utc
```

#### Uncomment `en_GB.UTF-8 UTF-8`: (Adjust accordingly)
```shell
nano /etc/locale.gen

# uncomment 
en_GB.UTF-8 UTF-8

# Generate locale:
locale-gen
```

#### Set system locale: (Adjust accordingly)
```shell
localectl set-locale LANG="en_GB.UTF-8"
localectl set-locale LC_TIME="en_GB.UTF-8"
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

#### Verify:
```shell
swapon --show
```

#### Ensure it's in fstab:
```shell
echo '/dev/vg/swap none swap defaults,discard 0 0' >> /etc/fstab
```
`discard` enables TRIM (only if your SSD supports it and `issue_discards = 1` in `/etc/lvm/lvm.conf`)

### 6.0 Initramfs Configuration
#### Edit `/etc/mkinitcpio.conf`:
```shell
nano /etc/mkinitcpio.conf
```

#### Ensure the `lvm2` hook is in the correct order:
```conf
HOOKS=(base udev autodetect microcode modconf kms keyboard keymap consolefont block lvm2 filesystems fsck)
```

#### Rebuild initramfs:
```shell
mkinitcpio -P
```

### 7.0 Enable Networking Services
```shell
systemctl enable dhcpcd
systemctl enable iwd.service
systemctl enable sshd        # Enable if you installed openssh
```

### 8.0 Install Microcode
#### For AMD:
```shell
pacman -S amd-ucode
```

#### For Intel:
```shell
pacman -S intel-ucode
```

#### Rebuild initramfs again:
```shell
mkinitcpio -p linux
```

### 9.0 Install Graphics Drivers
#### Intel/AMD:
```shell
pacman -S mesa
```

#### Enable Multilib (for 32-bit apps, Steam, etc.)
##### Uncomment in `/etc/pacman.conf`:
```conf
[multilib]
Include = /etc/pacman.d/mirrorlist
```

##### Update:
```shell
pacman -Syu
```

### Required to install AUR packages (optional)
```shell
pacman -S base-devel git
```

### 10.0  Set Root Password
```shell
passwd
```

### 11.0 Add User
```shell
useradd -m -G wheel,storage,power -s /bin/bash yourusername
passwd yourusername
```

### 12.0 Install `opendoas`
```shell
pacman -S opendoas
```

#### Allow user to run commands as root:
```shell
echo "permit persist keepenv yourusername" > /etc/doas.conf
chmod 600 /etc/doas.conf
```

#### Optional: Add `sudo` alias:
```shell
echo "alias sudo=doas" >> /home/yourusername/.bashrc
```

### 13.0 Install `limine`
```shell
pacman -S limine
```

### 14.0 Install `limine` Bootloader
```shell
mkdir -p /boot/EFI/BOOT
cp /usr/share/limine/BOOTX64.EFI /boot/EFI/BOOT/
```

### Create `/boot/limine.conf`
```shell
nano /boot/limine.conf
```

#### replace (`xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`... with actual UUID):
```conf
timeout: 5

/Arch Linux (linux)
    protocol: linux
    path: boot:/vmlinuz-linux
    module_path: boot:/amd-ucode.img        # Remove if Intel
    module_path: boot:/intel-ucode.img      # Remove if AMD
    module_path: boot:/initramfs-linux.img
    cmdline: root=/dev/vg/root rw rootfstype=ext4 add_efi_memmap vsyscall=none

/Arch Linux (linux-fallback)
    protocol: linux
    path: boot:/vmlinuz-linux
    module_path: boot:/amd-ucode.img        # Remove if Intel
    module_path: boot:/intel-ucode.img      # Remove if AMD
    module_path: boot:/initramfs-linux-fallback.img
    cmdline: root=/dev/vg/root rw rootfstype=ext4 add_efi_memmap vsyscall=none
```

### 15,0  Fix `/boot` Permissions
```shell
chmod 755 /boot
chmod 600 /boot/limine.conf
```

### 16.0 Secure `/tmp` Mount Options
#### Edit `/etc/fstab`:
```
nano /etc/fstab
```
#### Find the `/tmp` line and update options:
```shell
/dev/vg/tmp    /tmp    ext4    defaults,noatime,nosuid,nodev,auto    0 0
```
Prevents execution, device files, and suid abuse on `/tmp`

### 17.0 (Optional) Clear `/tmp` on Boot
```shell
echo "D /tmp 1777 root root 1d" > /etc/tmpfiles.d/clean-tmp.conf
```
Clears files older than 1 day. Use `0` instead of `1d` to wipe every boot.

## Randomize MAC Address (Privacy Enhancement)
### 1.0 Configure iwd for MAC Randomization
#### Create the global configuration file:
```shell
mkdir -p /var/lib/iwd
nano /var/lib/iwd/main.conf
```

#### Add the following content:
```shell
[General]
# Enable automatic network configuration (e.g., IP assignment)
EnableNetworkConfiguration=true

# Use systemd-resolved for DNS (if enabled later)
NameResolvingService=systemd

[Network]
# Prevent probing for default route on every network
DisableDefaultRoute=false

# Optional: Prevent DNS from being set by DHCP
# DisableDNSFromDHCP=true

[Device]
# Randomize MAC address using a stable per-network key (RECOMMENDED)
AddressRandomization=stable

# ALTERNATIVE: Randomize MAC once per boot (less private but more compatible)
# AddressRandomization=once

# OPTIONAL: Disable Wi-Fi power saving (can improve stability)
# PowerSave=false
```

#### Explanation of `AddressRandomization` options:

- `stable`: Generates a unique, persistent randomized MAC per network (SSID). Best balance of privacy and usability.

- `once`: Uses a new random MAC at each boot, same across all networks until reboot.

- `disabled`: Uses your real hardware MAC (not recommended for privacy).

✅ I recommend using `stable` unless you have compatibility issues.

### 2.0 Restart iwd to Apply Changes
```shell
systemctl restart iwd
```

### 3.0 Verify MAC Randomization
#### Check your wireless interface (replace `wlan0` if different):
```shell
ip link show wlan0
```
Look for the `ether` field:

- If randomized, it will not match your hardware (burned-in) MAC.
- The permanent MAC can be found with:

```shell
cat /sys/class/net/wlan0/address_mask
```

🛑 Troubleshooting Tip: If connection fails after enabling randomization, try switching to `once` and/or temporarily disable it. 

### 4.0 Reconnect to Wi-Fi (If Needed)
```shell
iwctl station wlan0 disconnect
iwctl station wlan0 connect YOUR_SSID
```
Note: Some networks (e.g., enterprise, captive portals) may require consistent MAC addresses. Adjust AddressRandomization as needed

##  DNS + Filtering (Recommended)
### #Step 1: Install Required Packages
```shell
doas pacman -S unbound curl wget
```

### Step 2: Initialize DNSSEC Root Key
```shell
doas unbound-anchor -a /etc/unbound/root.key
```
This enables DNSSEC validation for all queries. 

### Step 3: Configure Unbound with Local Filtering + DoT
#### Replace `/etc/unbound/unbound.conf`:
```shell
doas nano /etc/unbound/unbound.conf
```

#### Full Configuration (with Local Filtering & DoT)
```shell
server:
    # Listen locally
    interface: 127.0.0.1
    access-control: 127.0.0.1/32 allow

    # Privacy & security
    hide-identity: yes
    hide-version: yes
    use-caps-for-id: yes
    qname-minimisation: yes
    edns-cookie: yes

    # DNSSEC
    val-log-level: 1
    harden-dnssec-stripped: yes
    trust-anchor-file: /var/lib/unbound/root.key

    # Caching
    cache-max-ttl: 86400
    msg-cache-size: 128m

    # Forward to multiple encrypted providers
    forward-zone:
        name: "."
        forward-tls-upstream: yes

        # Primary: Quad9 (malware blocking, no logs)
        forward-addr: 9.9.9.9@853#dns.quad9.net

        # Secondary: Quad9 backup
        forward-addr: 149.112.112.112@853#dns.quad9.net
```

### Step 4: Add Local Filtering (Block Ads, Trackers, Malware)
#### We'll use the OISD blocklist, one of the most reliable and well-maintained ad/tracker blocklists.
```shell
doas nano /usr/local/bin/update-blocklist.sh
```

```
#!/bin/bash
BLOCKLIST_URL="https://oisd.nl/domainsonly"
OUTPUT="/etc/unbound/adblock.conf"

echo "# OISD Ad/Tracker Blocklist - $(date)" > $OUTPUT

while IFS= read -r domain; do
    case "$domain" in
        \#*|"") continue ;;
        *)
            echo "local-zone: \"$domain\" reject" >> $OUTPUT
            ;;
    esac
done < <(curl -s $BLOCKLIST_URL)

echo "✅ Blocklist updated from official OISD source: $OUTPUT"
doas systemctl reload unbound
```

#### Make executable and run:
```shell
doas chmod +x /usr/local/bin/update-blocklist.sh
doas /usr/local/bin/update-blocklist.sh
```
This creates `/etc/unbound/adblock.conf` with thousands of ad/tracker domains blocked locally. 

### Step 5: Automate Blocklist Updates (Daily)
```shell
doas systemctl edit --force --full unbound-blocklist.timer
```
#### Create Timer:
```shell
[Unit]
Description=Update Unbound Ad Blocklist Daily

[Timer]
OnCalendar=daily
Persistent=true

[Install]
WantedBy=timers.target
```

```shell
doas systemctl enable unbound-blocklist.timer
doas systemctl start unbound-blocklist.timer
```

### Step 7: Point System to Local Resolver
```shell
doas systemctl enable systemd-resolved
doas systemctl start systemd-resolved
```

#### Symlink:
```shell
doas ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf
```

#### Ensure `iwd` doesn't override DNS:
```
# /var/lib/iwd/main.conf
[Network]
DisableDNSFromDHCP=true
```

### Step 8: Verify Everything Works
#### Test normal resolution:
```shell
dig @127.0.0.1 google.com +short
```

#### Test blocking:
```shell
dig @127.0.0.1 doubleclick.net
```
Should return `NXDOMAIN` or no answer.

#### Test DNSSEC failure:
```shell
dig @127.0.0.1 dnssec-failed.org
```
Should return `SERVFAIL` if DNSSEC is working.

#### Check logs:
```shell
journalctl -u unbound -f
```

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
lsblk                 # Confirm /var, /tmp, swap LVs
swapon --show         # Verify swap active
cat /boot/limine.conf # Confirm bootloader config
cat /etc/fstab        # Ensure no cryptdevice entries
```

## You Now Have:
✅ Standard installation

✅ Separate `/`, `/home`, `/var`, `/tmp` via LVM

✅ LVM-based swap volume

✅ `limine` bootloader (no Secure Boot)

✅ Hardened `/tmp` with `nodev,nosuid`

✅ UTC time, proper locales, networking

✅ MAC address randomization for privacy

✅ Encrypted, Filtered DNS

✅ Minimal, maintainable base
