---
title: "System Commands"
description: "Instructions for bypassing Microsoft account setup and repairing system files using DISM and SFC."
date: 2025-11-26
tags: []
parent: Windows
---

# System Commands

## Bypass the Microsoft Account Setup

**Bypass the Microsoft Account setup during OOBE (out of box experience).**

During the "Let's add your Microsoft account" screen, press:
- `Shift` + `F10` to prompt a black terminal screen.
- This may be `Shift` + `Fn` + `F10` if you have a laptop with an `Fn` key.
**Tip**: Plug in an external keyboard if you're facing technical difficulties (laptop users).
- In the empty terminal type:
  ```
  start ms-cxh:localonly
  ```

{: .warning }
Pressing enter will take you to the local account creation screen.

***

## DISM and SFC Guide

**Repair corrupted system files using DISM and SFC.**

> DISM (Deployment Image Servicing and Management): Repairs the Windows component store`WinSxS`, which contains all the necessary restoration files; by default, it downloads files from Windows Update.

> SFC (System File Checker): Scans protected system files and replaces missing and/or corrupted files using the local cache in the component store.

### How to Run a Repair

- Press `Superkey` + `X` then `A`
  - The `Superkey` is often the `Windows` key 
- Accept the **UAC** (User Access Control) prompt
- Run these commands **in order**:
  ```
  DISM.exe /Online /Cleanup-Image /RestoreHealth
  ```
  ```
  SFC /ScanNow
  ```

{: .warning }
This DISM command requires an active internet connection.  
It uses `/Online` to download valid system files.

***

## Disable Fast Startup

**Disable Fast Startup and Hibernation.**

- Press `Superkey` + `X` / `Windows Icon` + `X` followed by `A`
- Accept the **(UAC)** prompt
- Disable Fast Startup:
  ```
  powercfg /h off
  ```

{: .warning }
**Note:**
Disabling hibernation removes "hiberfil.sys," freeing up several gigabytes of disk space in proportion to the users' RAM and ensures a complete system shutdown, reducing malware persistence across restarts and other system-related issues.