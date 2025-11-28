---
title: "System Logs"
description: "Guide to collecting and analyzing Microsoft Defender and process logs on Windows."
date: 2025-11-26
tags: []
parent: Windows
---

# System Logs

## Microsoft Defender Log Guide

**Microsoft Defender Logs**

Head to: `C:\ProgramData\Microsoft\Windows Defender\Support\`
- Upload the following `.txt` logs here for further analysis:
```
MPDectection
MPLog
```

***

## Process list Guide

**Upload your process list**

- Press `Superkey` + `X` / `Windows Icon` + `X` followed by `A`
- Accept the **(UAC)** prompt
- Type the following: `Get-Process > Documents\proclist.txt`
- Open file explorer > **Documents**, upload `proclist.txt` here