---
title: "Debloat Setup Guide"
description: "Instructions for using autounattend.xml and removing unnecessary Windows components."
date: 2025-11-26
tags: []
parent: Debloat
---

# Debloat Setup Guide

We have prepared an `autounattend.xml` which can be used when installing Windows to prevent bloat from being installed. You can view or download it below:
 
  <a href="autounattend.xml" class="btn btn-primary">View autounattend.xml</a>  
  <a href="autounattend.xml" download="autounattend.xml" class="btn btn-secondary">Download autounattend.xml</a>

---

## How to Use

**Option 1:**  
- Burn `autounattend.xml` into your Windows 11 ISO with [PowerISO](https://www.poweriso.net/PowerISO9-x64.exe) to create bootable media.


**Option 2:**  
- Mount your Windows 11 ISO by right-clicking it and selecting "Mount." 
- Copy `autounattend.xml` into the mounted drive.
- Launch `setup.exe` to begin installation.

---

## What's Removed?
This `autounattend.xml` file prevents the following "features" from being installed:
```text
3D Viewer
Internet Explorer
Movies & TV
Office 365
Photos
Solitaire Collection
Sticky Notes
To Do
Weather
Windows Media Player (modern)
Bing Search
Clipchamp
Cortana
Family
Get Help
Mail and Calendar
News
OneDrive
Power Automate
Quick Assist
Skype
Voice Recorder
Your Phone / Phone Link
Feedback Hub
Maps
OneNote
People
Recall
Tips
Wallet
```

{: .important }
This setup is intended for Intel/AMD 64-bit processor architectures.