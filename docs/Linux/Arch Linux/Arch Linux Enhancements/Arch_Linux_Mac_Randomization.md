---
title: "Arch Linux MAC Randomization"
description: "Guide to configuring MAC address randomization for privacy on Arch Linux using iwd."
date: 2025-11-26
tags: []
parent: Arch Linux Enhancements
---

## Privacy: Randomize MAC Address
### 1.0 Configure `iwd` for MAC Randomization
#### Create the global configuration file:
```shell
mkdir -p /etc/iwd
nano /etc/iwd/main.conf
```

#### Add the following content:
```shell
[General]
# Enable automatic network configuration (e.g., IP assignment)
EnableNetworkConfiguration=true

# Use systemd-resolved for DNS (if enabled later)
NameResolvingService=systemd

[Network]
# Disable automatic probing for default gateway on every network (recommended)
DisableDefaultRoute=true

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
Note: `DisableDefaultRoute=true` means the system will not automatically set a default gateway when connecting to a network. This enhances privacy on public Wi-Fi by reducing exposure. 

#### Explanation of `AddressRandomization` options:

- `stable`: Uses a unique, persistent randomized MAC derived from a hash of the SSID. Provides strong privacy while avoiding re-authentication on known networks.

(Default in many modern distros like Fedora, Ubuntu)

- `once`: Assigns a new random MAC at each boot, used across all networks until reboot. Less private than `stable`, but more compatible.

- `disabled`: Uses your real hardware (burned-in) MAC address — not recommended for privacy.

Recommendation: Use `stable` unless you encounter connectivity issues.

### 2.0 Restart `iwd` to Apply Changes
```shell
systemctl restart iwd
```

### 3.0 Verify MAC Randomization
#### Check the current MAC address:
```shell
ip link show wlan0
```
Replace `wlan0` with your actual wireless interface (e.g., `wlp2s0`, `wifi0`). 

#### Look for the `ether` field in the output:
```shell
link/ether xx:xx:xx:xx:xx:xx brd ...
```
- If MAC randomization is active, this address will differ from your hardware (burned-in) MAC.

- It should not match your device’s original factory-assigned MAC.

#### Find the permanent (hardware) MAC address:
```shell
ethtool -P wlan0
```
Replace `wlan0` with your actual wireless interface name (e.g., `wlp2s0`, `wifi0`, etc.). 

Example output:
```shell
Permanent address: aa:bb:cc:dd:ee:ff
```

If `ethtool` is not installed:
```shell
doas pacman -S ethtool
```
Verification: Compare the `Permanent address` (from `ethtool`) with the `ether` address (from `ip link`).
If they are different, MAC randomization is working.

- Note: On VMs or certain hardware, the addresses may appear identical — this could mean randomization is not supported or failed. 

### 4.0 Reconnect to Wi-Fi (If Needed)
#### If you were already connected, disconnect and reconnect to apply the new MAC:
```shell
iwctl station wlan0 disconnect
iwctl station wlan0 connect YOUR_SSID
```
Replace `YOUR_SSID` with your Wi-Fi network name.

You may be prompted to re-enter the password. 

### Important Notes

- `stable` mode: A unique random MAC is generated per SSID. Balances privacy and convenience.

#### Some networks may block randomized MACs:

- Enterprise networks (corporate, school)

- Captive portals (hotels, airports)

- Networks that bind access to your MAC address

#### In such cases, temporarily change the config:
```shell
[Device]
AddressRandomization=once
```

or

```
[Device]
AddressRandomization=disabled
```
then reconnect to the network.
