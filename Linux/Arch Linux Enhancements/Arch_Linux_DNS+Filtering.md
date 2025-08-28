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