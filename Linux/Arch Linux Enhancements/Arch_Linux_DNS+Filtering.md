##  DNS + Filtering (Recommended)
### Step 1.0: Install Required Packages
```shell
doas pacman -S unbound curl wget
```

### Step 2.0: Initialize DNSSEC Root Key
```shell
doas unbound-anchor -a /etc/unbound/root.key
```
This enables DNSSEC validation for all queries. 

### Step 3.0: Configure Unbound with Local Filtering + DoT
#### Backup default config (optional):
```shell
doas cp /etc/unbound/unbound.conf /etc/unbound/unbound.conf.bak
```

#### Edit the main configuration file:
```shell
doas nano /etc/unbound/unbound.conf
```

```shell
include: /etc/unbound/adblock.conf

server:
    # Listen locally
    interface: 127.0.0.1
    access-control: 127.0.0.1/32 allow

    # IPv6 (optional, if enabled on system)
    # interface: ::1
    # access-control: ::1/128 allow

    # Enhance privacy
    hide-identity: yes
    hide-version: yes
    use-caps-for-id: yes
    qname-minimisation: yes
    edns-cookie: yes

    # Improve performance
    prefetch: yes
    prefetch-key: yes
    cache-max-ttl: 86400
    msg-cache-size: 128m

    # Enforce DNSSEC
    val-log-level: 1
    harden-dnssec-stripped: yes
    trust-anchor-file: /etc/unbound/root.key

    # Forward queries over encrypted DNS (DoT)
    forward-zone:
        name: "."
        forward-tls-upstream: yes
        forward-addr: 9.9.9.9@853           # Quad9 (malware blocking)
        forward-addr: 149.112.112.112@853   # Quad9 backup

    # Log blocked domains (optional for debugging)
    log-local-actions: yes
```
Quad9 blocks known malware domains by default.

Use `journalctl -u unbound -f | grep "local_data"` to monitor blocked domains.

### Step 4.0: Add Local Filtering (Block Ads, Trackers, Malware)
#### We'll use the OISD blocklist, one of the most reliable and well-maintained ad/tracker blocklists.
```shell
doas nano /usr/local/bin/update-blocklist.sh
```

```shell
#!/bin/bash

set -euo pipefail

BLOCKLIST_URL="https://oisd.nl/domainsonly"
OUTPUT="/etc/unbound/adblock.conf"
TEMP_OUTPUT="${OUTPUT}.tmp"

echo "Fetching blocklist from: $BLOCKLIST_URL"

DOMAINS=$(curl -sf --max-time 30 "$BLOCKLIST_URL")
if [ -z "$DOMAINS" ]; then
    echo "ERROR: Failed to fetch blocklist. Check URL or network."
    exit 1
fi

{
    echo "# OISD Ad/Tracker Blocklist - $(date -u)"
    echo "# Source: $BLOCKLIST_URL"
    echo ""
} > "$TEMP_OUTPUT"

while IFS= read -r domain; do
    case "$domain" in
        \#*|"") continue ;;
        *)
            # Basic validation: alphanumeric, dots, hyphens, underscores only
            if [[ "$domain" =~ ^[a-zA-Z0-9._-]+$ ]] && [ ${#domain} -le 253 ]; then
                printf 'local-zone: "%s" redirect\nlocal-data: "%s A 0.0.0.0"\n' "$domain" "$domain"
            else
                echo "Skipping invalid domain: $domain" >&2
            fi
            ;;
    esac
done <<< "$DOMAINS" >> "$TEMP_OUTPUT"

if [ ! -s "$TEMP_OUTPUT" ]; then
    echo "ERROR: Generated blocklist is empty."
    rm -f "$TEMP_OUTPUT"
    exit 1
fi

# Validate config before applying
if ! doas unbound-checkconf; then
    echo "ERROR: Current Unbound config is invalid. Aborting update."
    rm -f "$TEMP_OUTPUT"
    exit 1
fi

doas mv "$TEMP_OUTPUT" "$OUTPUT"
doas chown root:unbound "$OUTPUT"
doas chmod 644 "$OUTPUT"

echo "Blocklist updated: $OUTPUT"
doas systemctl reload unbound

echo "Done. $(grep -c '^local-zone' "$OUTPUT") domains blocked."
```

#### Make script executable:
```shell
doas chmod +x /usr/local/bin/update-blocklist.sh
```

### Step 5.0: Automate Blocklist Updates with systemd
#### Create service unit:
```shell
doas systemctl edit --force --full unbound-blocklist.service
```

```shell
[Unit]
Description=Update Unbound Ad Blocklist
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/update-blocklist.sh
User=root
RemainAfterExit=yes
```

#### Create timer unit:
```shell
doas systemctl edit --force --full unbound-blocklist.timer
```

```shell
[Unit]
Description=Update Unbound Ad Blocklist Daily
After=network-online.target
Wants=network-online.target

[Timer]
OnCalendar=daily
Persistent=true
RandomizedDelaySec=1h

[Install]
WantedBy=timers.target
```
`RandomizedDelaySec=1h` prevents server load spikes.

#### Enable and start timer:
```shell
doas systemctl daemon-reload
doas systemctl enable --now unbound-blocklist.timer
```

#### Check timer status:
```shell
systemctl list-timers | grep unbound
```

### Step 6.0: Configure System DNS to Use Unbound
```shell
doas nano /etc/resolv.conf
```

#### Replace the contents with:
```shell
nameserver 127.0.0.1
# nameserver ::1     # Uncomment if using IPv6
options edns0
```
This file may be overwritten by DHCP or network managers.

#### Prevent overwrites:
```shell
doas chattr +i /etc/resolv.conf
```
Revert with `doas chattr -i /etc/resolv.conf` if needed. 

#### Ensure `iwd` doesn't override DNS:
```shell
doas nano /etc/iwd/main.conf
```

#### And ensure it contains:
```shell
[Network]
DisableDNSFromDHCP=true
```

#### restart iwd:
```shell
doas systemctl restart iwd
```

### Step 7.0: Start and Enable Unbound
```shell
doas systemctl enable --now unbound
```

### Step 8.0: Verify Everything Works
#### Test normal resolution:
```shell
dig @127.0.0.1 google.com +short
```

#### Test blocking:
```shell
dig @127.0.0.1 doubleclick.net
# Should return: 0.0.0.0
```

#### Test DNSSEC failure:
```shell
dig @127.0.0.1 dnssec-failed.org
# Should return: status: SERVFAIL
```

#### Review live logs:
```shell
journalctl -u unbound -f --since "1 min ago"
```

#### Check blocklist timer:
```shell
systemctl list-timers | grep unbound
```