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

### uBlock Origin Guide

**Ungoogled Chromium uBlock Origin Guide**
- Download [uBlock Origin](<https://github.com/gorhill/uBlock/releases>) here by selecting `uBlock0_x.xx.x.chromium.zip` under assets
- Head to `chrome://extensions`, check mark `Developer mode` top right
- Unzip `uBlock0_x.xx.x.chromium.zip`
- Load unpacked > Select `uBlock0.chromium
