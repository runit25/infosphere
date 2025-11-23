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

DISM, short for Deployment Image Servicing and Management, is a command-line tool provided by Microsoft to service Windows images (WIM, VHD, and VHDX files) and prepare them for deployment, which is achieved by downloading the replacement files from Windows servers.

SFC, short for System File Checker. Scans for important system files and restores them using the cached copy located within the compressed folder of Windows itself.

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