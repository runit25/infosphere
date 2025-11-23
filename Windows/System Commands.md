# System Commands

## Bypass the Microsoft Account Setup During OOBE

**Bypass the Microsoft Account setup during OOBE.**

During the "Let's add your Microsoft account" screen, press:

- `Shift` + `F10` / `Shift` + `Fn` + `F10` to prompt a black terminal screen.
- In the empty terminal type:
  ```
  start ms-cxh:localonly
  ```
  - __Note:__  Pressing enter will take you to the local account creation screen.

***

## DISM and SFC Guide

**Repair corrupted system files using DISM and SFC.**

DISM (Deployment Image Servicing and Management): Repairs the Windows component store`WinSxS`, which contains all the necessary restoration files; by default, it downloads files from Windows Update.

SFC (System File Checker): Scans protected system files and replaces missing and/or corrupted files using the local cache in the component store.

- Press `Superkey` + `X` / `Windows Icon` + `X` followed by `A`
- Accept the **(UAC)** prompt
- Run these commands **in order**:
  ```
  DISM.exe /Online /Cleanup-Image /RestoreHealth
  ```
  ```
  SFC /ScanNow
  ```
  - __Note:__  "DISM" requires an active internet connection.

***

## Disable Fast Startup

**Disable Fast Startup.**

- Press `Superkey` + `X` / `Windows Icon` + `X` followed by `A`
- Accept the **(UAC)** prompt
- Disable Fast Startup:
  ```
  powercfg /h off
  ```
  - __Note:__  Disabling hibernation removes "hiberfil.sys," freeing up several gigabytes of disk space in proportion to the users' RAM. and ensures a complete system shutdown on reboot, reducing malware persistence across restarts. and other system-related issues.