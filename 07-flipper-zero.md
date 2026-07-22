# 07 — Flipper Zero

The Flipper Zero is a portable multi-tool for hardware hacking and radio protocols — Sub-GHz, NFC, RFID, infrared, iButton, and BadUSB in one device. This guide covers replacing the stock firmware with Momentum, a community firmware that adds a significantly larger set of features and apps. The Flipper is not connected to the Pi, but it is documented here as part of the wider homelab setup.

---

## Why Momentum

The official firmware is conservative by design — it ships with regional restrictions on Sub-GHz frequencies and a limited app catalogue. Momentum removes those restrictions, adds a large number of built-in apps, and introduces a theming system (Asset Packs) that lets you customise every visual element of the device. It is a direct continuation of the Xtreme firmware by the same developers. [22] [23]

---

## Before you start

- Your saved files (Sub-GHz captures, NFC cards, remotes, etc.) are stored on the SD card, not in the Flipper's internal storage. They are not touched by a firmware update.
- The qFlipper desktop app and the Web Updater cannot be open simultaneously — close qFlipper before proceeding.
- The Web Updater requires a Chromium-based browser (Chrome, Edge, Brave). It does not work in Firefox.

---

## Installing Momentum

The Web Updater is the simplest install method and requires no file downloads.

**1 —** Connect your Flipper Zero to your computer via USB.

**2 —** Open `https://momentum-fw.dev/update` in Chrome (or another Chromium-based browser).

**3 —** Click **Connect** and select your Flipper from the popup.

**4 —** Choose a channel — **Release** for stability, **Dev** for the latest features. Release is the right choice for most people.

**5 —** Click **Flash** and wait for the update to complete. Do not disconnect the Flipper during this process.

The Flipper will reboot into Momentum when the flash is done.

---

## After installing

**Flipper Lab for apps:** After installing Momentum, the official Flipper mobile app will no longer work for installing applications. Use Flipper Lab at `https://lab.flipper.net` in Chrome instead — it connects via USB and lets you browse and install the full app catalogue.

**Asset Packs:** Momentum includes a theming system that lets you change the Flipper's animations, icons, and fonts. Packs can be downloaded from the Momentum website or community Discord and uploaded to `SD/asset_packs/` via the File Manager tab in qFlipper. To apply a pack, press the Up arrow on the Flipper home screen, go to **Momentum Settings → Interface → Graphics → Asset Pack**.

**Updating:** Check `https://momentum-fw.dev` for new releases. To update, repeat the Web Updater process — your SD card data is preserved.

---

**Next:** [08 — Docker](08-docker.md)

**Sources:** [22] [23]