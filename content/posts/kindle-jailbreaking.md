+++
title = "Kindle Jailbreaking: Unlocking Koreader and Custom Extensions"
date = 2022-10-02T12:17:25-07:00

[taxonomies]
categories = ["Hacking"]
tags = ["kindle", "jailbreak", "koreader", "e-ink"]
+++

Amazon's Kindle firmware locks you into the stock reader, which handles PDFs poorly and limits customization. Jailbreaking opens access to Koreader (a reader with PDF reflow and custom dictionaries), KUAL (a plugin launcher), and system-level tools like SSH. This guide covers the demo mode exploit for firmware 5.14.2 and earlier.

<!--more-->

**Update (July 2026):** This guide was written in 2022 and covers the 5.14.2 demo mode exploit. For newer Kindle models (11th gen Paperwhite, Kindle Scribe) and later firmware versions, see [KindleModding.org](https://kindlemodding.org) — they maintain an actively updated wiki with methods for recent devices.

## Prerequisites

**Compatible devices:** Kindle Paperwhite (7th-10th gen), Kindle Oasis (8th-10th gen), Kindle Basic (8th-10th gen). Check your firmware version under Settings → Device Info.

**Risk:** Jailbreaking voids the warranty and can brick the device if interrupted during critical steps. Amazon does not support jailbroken Kindles. Back up your library before starting.

## Why Demo Mode

Kindle's demo mode is a retail display feature that runs unsigned code to showcase device capabilities. Firmware 5.14.2 and earlier do not verify signatures on files placed in the `.demo` directory during demo mode initialization. The exploit runs a script from that directory to gain root access. Amazon patched this in 5.14.3 by adding signature checks.

## Jailbreak Steps

Works only on firmware 5.14.2 or earlier.

1. **Upgrade to 5.14.2**  
   Download the [official 5.14.2 firmware](https://www.amazon.com/gp/help/customer/display.html?nodeId=GKMQC26VQQMM8XSW) for your model. Copy the `.bin` file to the Kindle's root directory. Go to Menu → Settings → Device Options → Advanced Options → Update Your Kindle. The device will reboot.

2. **Factory reset.**  
   Go to Menu → Settings → Device Options → Reset.

3. **Initial setup.**  
   Choose **English (United Kingdom)** as the language. Skip WiFi.

4. **Enter demo mode.**  
   Open the search box and type `;demo`. The Kindle will switch to demo mode.

5. **Reboot.**  
   Go to Menu → Settings → Device Options → Restart.

6. **Install the jailbreak payload.**  
   - Download [watchthis-jailbreak-r03.zip](https://mega.nz/file/2ahlQKZS#jXyYLEp9rvRQCOzv7LNYBF-9fOfPhpigaLZMHZkN7fg). The string after `#` is the decryption key.
   - Extract and create a directory named `.demo` at the root of Kindle storage.
   - Copy the files from the zip into `.demo` as instructed in the [Katadelos thread](https://www.mobileread.com/forums/showthread.php?t=346037).
   - Eject and reboot. The jailbreak script runs during demo mode initialization.
   
   If the screen shows an error, reboot and retry.

7. **Install the hotfix.**  
   The [Jailbreak Hotfix](https://www.mobileread.com/forums/showpost.php?p=3004892&postcount=1597) hooks into the firmware update process before signature verification runs, preserving the jailbreak across updates. Install it via KUAL after you complete the Extensions section below.

## Updating Firmware After Jailbreak

Before updating to newer firmware:
1. Check the [hotfix thread](https://www.mobileread.com/forums/showpost.php?p=3004892&postcount=1597) for the latest version compatible with your target firmware.
2. Install the updated hotfix via KUAL.
3. Download the new firmware `.bin`, place it in the root directory, and trigger the update through Settings.

If KUAL is missing after the update, the hotfix failed and you must re-jailbreak from 5.14.2.

## Extensions

Install these in order.

**Core (install first):**

- **[MobileRead Package Installer (MRPI)](https://www.mobileread.com/forums/showthread.php?t=251143)** — Installs `.bin` plugin packages. Required for everything below.
- **[KUAL](https://www.mobileread.com/forums/showthread.php?t=203326)** — Plugin launcher. Adds a "KUAL" icon to the Kindle home screen that lists installed extensions.

**Essential:**

- **[Koreader](https://github.com/koreader/koreader)** — The reason most people jailbreak. Handles PDFs with reflow, supports ePub/DjVu/CBZ, offers configurable margins and fonts, and includes StarDict dictionary support.
- **[Jailbreak Hotfix](https://www.mobileread.com/forums/showpost.php?p=3004892&postcount=1597)** — Mentioned earlier. Install after KUAL so you can run the installer script.

**Optional:**

- **[Helper](https://www.mobileread.com/forums/showthread.php?t=203326)** — Utilities for screenshots, SSH server toggle, and font management.
- **[BatteryStatus](https://www.mobileread.com/forums/showpost.php?p=2636886&postcount=52)** — Displays battery percentage on the home screen.
- **[KUAL+](https://www.mobileread.com/forums/showpost.php?p=2591705&postcount=1014)** — Adds configuration options to KUAL (themes, menu tweaks).

Full extension lists: [MobileRead hacks index](http://www.mobileread.com/forums/showthread.php?t=225030), [snapshots thread](http://www.mobileread.com/forums/showthread.php?t=205064).

