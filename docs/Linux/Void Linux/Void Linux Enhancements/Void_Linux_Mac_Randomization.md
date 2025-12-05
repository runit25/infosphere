---
title: "Void Linux MAC Randomization"
description: "Guide to configuring MAC address randomization for privacy on Void Linux using iwd."
date: 2025-12-05
tags: []
parent: Void Linux Enhancements
---

## Randomize MAC Address
```shell
cat > /etc/iwd/main.conf <<EOF
[General]
EnableNetworkConfiguration=true

[Settings]
AlwaysRandomizeAddress=true
EOF
```
Note: Requires iwd