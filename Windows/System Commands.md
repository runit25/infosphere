# System Commands

## Microsoft Account Bypass

**Microsoft Account Bypass**

During the **Let's add your Microsoft account screen**

- Press `Shift` + `F10` / `Shift` + `Fn` + `F10` to prompt a **black terminal screen**
- Execute the following command:
```start ms-cxh:localonly```
Plug in an external keyboard if `Shift` + `Fn` + `F10` doesn't work and try again (laptop Users)

***

## What the DISM, SFC commands achieve

**Brief description of what the DISM, SFC commands achieve**

DISM, short for Deployment Image Servicing and Management, is a command-line tool provided by Microsoft to service Windows images (WIM, VHD, VHDX files) and prepare them for deployment, which is achieved by downloading the replacement files from Windows servers.

SFC, short for System File Checker. Scans for important system files and restores them using the cached copy located within the compressed folder of Windows itself.

***

## DISM, SFC guide

**Repair important files with the DISM, SFC commands**

- Press `Superkey` + `X` / `Windows Icon` + `X` followed by `A`
- Accept the **(UAC)** prompt
- Run the following commands: 

  ```DISM.exe /Online /Cleanup-Image /RestoreHealth```
  
  ```SFC /ScanNow```
  - __Note:__  **DISM** requires an active **NAT** (Internet connection to function)

***

## Disable/Enable Fast-Startup

**Turn on/off fast-startup**

- Press `Superkey` + `X` / `Windows Icon` + `X` followed by `A`
- Accept the **(UAC)** prompt
- Follow the following commands:
- Disable fast-startup: `powercfg /h off`
- Enable fast-startup: `powercfg /h on`