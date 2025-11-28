---
title: "Ungoogled Chromium Guide"
description: "Install and configure Ungoogled Chromium on Windows using winget and browser flags."
date: 2025-11-26
tags: []
parent: Setup Guides
---

# Ungoogled Chromium

## Install guide

**Install Ungoogled Chromium**

- Press `Superkey` + `X` / `Windows Icon` + `X` followed by `A`
- Accept the **(UAC)** prompt
- paste `winget search ungoogled-chromium` into the terminal to discover **Ungoogled Chromium's** package (ID) value `eloston.ungoogled-chromium`
- Install **Ungoogled Chromium** `winget install -e eloston.ungoogled-chromium`
- Package (ID) maintenance `winget upgrade --all` **upgrades everything**

***

### Web Store Guide

**Ungoogled_Chromium Web Store Guide**

- Paste `chrome://flags/#extension-mime-request-handling` into **Ungoogled Chromium's** search bar
- Adjust `Handling of extension MIME type requests` to `Always prompt for install`
- Select relaunch
- Download [Chromium.Web.Store.crx](<https://github.com/NeverDecaf/chromium-web-store/releases>) here
- Accept the **Browser Prompt**, which allows you to install [Extensions here](<https://chromewebstore.google.com/>)

***

### uBlock Origin Lite Guide

**uBlock Origin Lite Guide**

Download [uBlock Origin Lite](<https://chromewebstore.google.com/detail/ddkjiahejlhfcafbddmgiahcphecmpfh>) here by pressing **Add to Chrome** followed by the `add extension` prompt.
- Configure `uBlock Origin Lite` for optimal content blocking capabilities by selecting the puzzle icon top right and clicking on the uBlock Origin Lite icon followed by the `gear cog`.
- Adjust Default filtering mode to `Complete`
- Navigate to Filter lists and checkmark all the boxes above `Regions, languages`