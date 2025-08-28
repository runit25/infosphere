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