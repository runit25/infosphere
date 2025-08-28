# Arch Linux Install

<!-- Created by https://gitlab.com/runit25/infosphere -->

Installation includes full disk encryption (LUKS2 + LVM), limine bootloader (Secure Boot), ext4, and base system.

**Note:** Substitute `/dev/nvme0nX` with your corresponding drive (**Example:** "/dev/sda")

## Pre-installation
### 1.0 Verify UEFI Boot Mode
#### Ensure you're in UEFI mode:
```shell
ls /sys/firmware/efi/efivars
```
If the directory exists you're free to continue.

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
```

#### Example output:
```shell
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
nvme0n1     259:0    0 476.9G  0 disk 
├─nvme0n1p1 259:1    0     1G  0 part 
└─nvme0n1p2 259:2    0 475.9G  0 part 
```
We will be installing Linux on `nvme0n1`.

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

### 6.0 Encrypt Root Partition (LUKS2)
#### Select based on your hardware:
#### Modern System (4+ cores, 16GB+ RAM)
```shell
cryptsetup luksFormat \
  --type luks2 \
  --cipher aes-xts-plain64 \
  --key-size 512 \
  --hash sha512 \
  --pbkdf argon2id \
  --iter-time 7500 \
  --pbkdf-memory 1048576 \
  --pbkdf-parallel 4 \
  --sector-size 512 \
  --align-payload 8192 \
  --label arch_root_encrypted \
  /dev/nvme0n1p2
```

#### Mid-Range (2–4 cores, 8GB RAM)
```shell
cryptsetup luksFormat \
  --type luks2 \
  --cipher aes-xts-plain64 \
  --key-size 512 \
  --hash sha512 \
  --pbkdf argon2id \
  --iter-time 5000 \
  --pbkdf-memory 524288 \
  --pbkdf-parallel 2 \
  --sector-size 512 \
  --align-payload 8192 \
  --label arch_root_encrypted \
  /dev/nvme0n1p2
```

#### Low-End (2 cores, 2-4GB RAM)
```shell
cryptsetup luksFormat \
  --type luks2 \
  --cipher aes-xts-plain64 \
  --key-size 512 \
  --hash sha512 \
  --pbkdf argon2id \
  --iter-time 3000 \
  --pbkdf-memory 262144 \
  --pbkdf-parallel 1 \
  --sector-size 512 \
  --align-payload 8192 \
  --label arch_root_encrypted \
  /dev/nvme0n1p2
```
You'll be prompted for a passphrase.

### 7.0 Open Encrypted Volume
```shell
cryptsetup luksOpen /dev/nvme0n1p2 lukspart
```
This exposes the decrypted volume at `/dev/mapper/lukspart`.

### 8.0 Create LVM Physical Volume & Volume Group
```shell
pvcreate /dev/mapper/lukspart
vgcreate vg /dev/mapper/lukspart
```

### 9.0 Create Logical Volumes
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

### 10.0 Format Filesystems
```shell
mkfs.ext4 -L "Arch Root"   /dev/vg/root
mkfs.ext4 -L "Arch Var"    /dev/vg/var
mkfs.ext4 -L "Arch Tmp"    /dev/vg/tmp
mkfs.ext4 -L "Arch Home"   /dev/vg/home
mkswap /dev/vg/swap        # Format swap LV
```

### 11.0 Mount Filesystems
#### Mount root first:
```shell
# Mount root first
mount /dev/vg/root /mnt
```

#### Create and mount other directories:
```shell
mkdir -p /mnt/{home,var,tmp,boot}
mount /dev/vg/home /mnt/home
mount /dev/vg/var  /mnt/var
mount /dev/vg/tmp  /mnt/tmp
```

#### Format and mount EFI partition:
```shell
mkfs.fat -F32 /dev/nvme0n1p1
mount /dev/nvme0n1p1 /mnt/boot
```
`/boot` must remain unencrypted for UEFI boot. 

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

#### Expected `lsblk` output:
```shell
# output
NAME          MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINT
nvme0n1       259:0    0 476.9G  0 disk  
├─nvme0n1p1   259:1    0     1G  0 part  /boot
└─nvme0n1p2   259:2    0 475.9G  0 part  
  └─lukspart  254:0    0 475.9G  0 crypt 
    ├─vg-root 254:1    0    20G  0 lvm   /
    ├─vg-var  254:2    0    20G  0 lvm   /var
    ├─vg-tmp  254:3    0     8G  0 lvm   /tmp
    ├─vg-swap 254:4    0     4G  0 lvm   [SWAP]
    └─vg-home 254:5    0 423.9G  0 lvm   /home
```

### 3.0 Set Time and Locale
```shell
timedatectl set-ntp true
timedatectl set-timezone UTC # Avoids DST issues
hwclock --systohc --utc
```

#### Uncomment `en_GB.UTF-8 UTF-8` in `/etc/locale.gen`: (Adjust accordingly)
```shell
nano /etc/locale.gen
```

#### Generate and set locale: (Adjust accordingly)
```shell
locale-gen
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
#### Activate:
```shell
swapon /dev/vg/swap
```

#### Verify:
```shell
swapon --show
```

#### Ensure it's in `fstab`:
```shell
echo '/dev/vg/swap none swap defaults,discard 0 0' >> /etc/fstab
```
`discard` enables TRIM (only if your SSD supports it and `issue_discards = 1` in `/etc/lvm/lvm.conf`).

### 6.0 Initramfs Configuration
#### Edit `/etc/mkinitcpio.conf`:
```shell
nano /etc/mkinitcpio.conf
```

#### Ensure `hooks` include `encrypt` and lvm2 in order:
```conf
HOOKS=(base udev autodetect microcode modconf kms keyboard keymap consolefont block encrypt lvm2 filesystems fsck)
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

#### Enable Multilib (for 32-bit apps, Steam, etc.):
##### Uncomment in `/etc/pacman.conf`:
```conf
[multilib]
Include = /etc/pacman.d/mirrorlist
```

##### Update:
```shell
pacman -Syu
```

### Required to install AUR packages (optional):
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

### 12.0 12. Install `opendoas`
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

### 13.0 Install `limine` and `sbctl`
```shell
pacman -S limine sbctl
```

### 14.0 Generate Secure Boot Keys
```shell
sbctl generate-keys
```
Creates:

- Private: `/var/db/sbctl/owner.key`
- Public: `/var/db/sbctl/owner.crt`

### 15.0 Enroll Keys via MokManager
```shell
sbctl enroll-keys --mok
```
You'll be prompted for a PEM password — remember it!

#### On next boot:

1. Blue MokManager screen appears
2. Select "Enroll MOK"
3. "Continue" → "Yes"
4. Enter the password you set
After enrollment, disable Setup Mode in firmware.

### 16.0 Install `limine` Bootloader
```shell
mkdir -p /boot/EFI/BOOT
cp /usr/share/limine/BOOTX64.EFI /boot/EFI/BOOT/
```

### 17.0 Get LUKS Partition UUID
```shell
blkid -s UUID -o value /dev/nvme0n1p2
```
Remember the output (e.g., `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`).

### Create `/boot/limine.conf`
```shell
nano /boot/limine.conf
```

```conf
timeout: 5

/Arch Linux (linux)
    protocol: linux
    path: boot:/vmlinuz-linux
    module_path: boot:/amd-ucode.img        # Remove if Intel
    module_path: boot:/intel-ucode.img      # Remove if AMD
    module_path: boot:/initramfs-linux.img
    cmdline: cryptdevice=UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx:lukspart root=/dev/vg/root rw rootfstype=ext4 add_efi_memmap vsyscall=none

/Arch Linux (linux-fallback)
    protocol: linux
    path: boot:/vmlinuz-linux
    module_path: boot:/amd-ucode.img        # Remove if Intel
    module_path: boot:/intel-ucode.img      # Remove if AMD
    module_path: boot:/initramfs-linux-fallback.img
    cmdline: cryptdevice=UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx:lukspart root=/dev/vg/root rw rootfstype=ext4 add_efi_memmap vsyscall=none
```
replace (`xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`... with actual UUID):

### 19,0  Fix `/boot` Permissions
```shell
chmod 755 /boot
chmod 600 /boot/limine.conf
```

### 20.0 Sign Boot Files
```shell
sbctl sign -s /boot/EFI/BOOT/BOOTX64.EFI
sbctl sign -s /boot/vmlinuz-linux
sbctl sign -s /boot/initramfs-linux.img
sbctl sign -s /boot/initramfs-linux-fallback.img
sbctl sign -s /boot/amd-ucode.img   # Skip if Intel  
sbctl sign -s /boot/intel-ucode.img # Skip if AMD
```
#### Verify:
```shell
sbctl verify
```

### 21.0 Automate Signing on Updates
```shell
nano /etc/pacman.d/hooks/100-sign-secureboot.hook
```

```conf
[Trigger]
Type = Path
Operation = Upgrade
Target = boot://vmlinuz-linux
Target = boot://initramfs-linux.img
Target = boot://amd-ucode.img
Target = boot://intel-ucode.img

[Action]
Description = Signing EFI binaries for Secure Boot
When = PostTransaction
Exec = /usr/bin/sbctl sign-all
```

### 22.0 Secure `/tmp` Mount Options
#### Edit `/etc/fstab`:
```
nano /etc/fstab
```
#### Find the `/tmp` line and update options:
```shell
/dev/vg/tmp    /tmp    ext4    defaults,noatime,nosuid,nodev,auto    0 0
```
Prevents execution, device files, and suid abuse on `/tmp`

### 23.0 (Optional) Clear `/tmp` on Boot
```shell
echo "D /tmp 1777 root root 1d" > /etc/tmpfiles.d/clean-tmp.conf
```
Clears files older than 1 day. Use 0 to wipe every boot.

## Privacy: Randomize MAC Address
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

### 2.0 Restart `iwd` to Apply Changes
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

## Post Reboot

#### 1. Enroll MOK Key
- Blue MokManager screen appears
- Follow prompts to enroll key using the password you set
- After success, disable Setup Mode in UEFI settings

#### 2. Enable Secure Boot
In UEFI Firmware:

- Secure Boot: `Enabled`
- Mode: User Mode or `Custom Mode`
- Setup Mode: `Disabled`

## Verify Installation
#### After logging in:
```shell
mokutil --sb-state            # Should say "Secure boot enabled"
sbctl status                  # Should show enrolled keys and signed binaries
lsblk                         # Confirm /var, /tmp, swap LVs
swapon --show                 # Verify swap active
dmesg | grep -i "secure boot" # Confirm kernel sees Secure Boot
```

## You Now Have:
✅ Full disk encryption (LUKS2 + LVM)

✅ Isolated `/`, `/home`, `/var`, `/tmp`

✅ LVM-based swap

✅ `limine` with Secure Boot

✅ Automated kernel signing (`sbctl`)

✅ Hardened `/tmp` (`nodev,nosuid`)

✅ UTC time, correct locales, networking

✅ MAC address randomization

✅ Encrypted, filtered DNS

✅ Minimal, secure, maintainable base
