+++
title = "iPhone ICCID Carrier Unlock: Activation Ticket Backup and Restore"
date = 2026-06-22T00:00:00+00:00

[taxonomies]
categories = ["Device"]
tags = ["devices", "security", "iOS"]
+++

ICCID activation was once a popular way to work around carrier locks on iPhones. If a previously activated device is jailbreakable, its activation ticket can be backed up and restored after a reset, iOS downgrade, or DFU restore. The ticket is device-specific and only preserves activation on the iPhone from which it was saved. Here I use my iPhone 5s (A1453, au/KDDI Japan model) as an example.

<!--more-->

## Background

### What is ICCID Unlock

Carrier-locked iPhones are bound to a specific carrier at the activation-server level. When you insert a SIM from a different carrier, the iPhone contacts Apple's activation server and may reject the SIM.

The ICCID method used a SIM interposer configured with a working ICCID to pass Apple's activation check. Once activated, the iPhone stored an `ActivationTicket` locally. Guides sometimes call this value an "unlock certificate," but **activation ticket** is the more precise term.

### Why This Still Matters

Apple has patched the ICCID activation vulnerability, so a new activation ticket can no longer be obtained through this method. Previously issued tickets were not revoked as part of the fix, however. If your device was activated before the patch and its ticket is still present, back it up before changing the system. A saved ticket can be restored after any of the following:

- Factory reset (Erase All Content and Settings)
- iOS upgrade/downgrade
- DFU restore

**Important:** The activation ticket is tied to the device identity. It only works on the original iPhone and cannot be transferred to another device.

### My Device

iPhone 5s model A1453, the au/KDDI Japan carrier-locked variant. This model supports CDMA + GSM with LTE Band 18/26 (au's primary bands). Previously ICCID unlocked, now running with a non-au SIM card.

The iPhone 5s (A7 chip) is particularly interesting for this purpose:
- Vulnerable to checkm8 bootrom exploit, so it's jailbreakable on any iOS version forever
- Apple still signs iOS 10.3.3 OTA for A7 devices, allowing downgrade without SHSH blobs
- Supports iOS 7 through [12.5.8](https://support.apple.com/en-us/100100), giving a wide range of jailbreak options

## Jailbreak

The jailbreak provides filesystem access to read/write the activation ticket. Two straightforward methods for the versions covered below:

### iOS 10.3.3: TotallyNotSpyware (Online Jailbreak)

[TotallyNotSpyware](https://lukezgd.github.io/tns) (TNS) is a browser-based jailbreak for all 64-bit devices on iOS 10.0–10.3.3. It uses the SockPort exploit to achieve kernel code execution directly from Safari, no computer needed.

**Steps:**

1. Open Safari on the iPhone, navigate to: `https://lukezgd.github.io/tns`

2. Slide the "Slide for Spyware" slider from left to right.

3. When the prompt appears, press "noot noot". Wait for the jailbreak to complete.
   - If the device reboots without showing the prompt, try again. The exploit doesn't always succeed on the first attempt.

4. After success, Zebra (package manager) appears on the home screen. You're jailbroken.

**Notes:**
- Semi-untethered: the jailbreak is lost after reboot. Revisit the TNS page to re-jailbreak.
- Tip: use Share → Add to Home Screen for quick access.
- Works entirely on-device with no PC required.

### iOS 12.0–12.5.8: Chimera

[Chimera](https://chimera.coolstar.org) is a semi-untethered jailbreak by Coolstar/Electra Team. Although its official site still labels the latest download as supporting iOS 12.0–12.5.7, [ios.cfw.guide](https://ios.cfw.guide/installing-chimera/) confirms that Chimera works on A7–A11 devices through iOS 12.5.8. For an iPhone 5s on iOS 12.4.1–12.5.8, the guide recommends an unofficial build with [chimera_patch](https://github.com/staturnzz/chimera_patch) for better success rates; the original build also works.

**Steps:**

1. Sideload the Chimera IPA using AltStore, Sideloadly, or TrollStore (if available on your iOS version).

2. Open the Chimera app on the home screen, tap "Jailbreak".

3. Wait for the device to respring. Sileo (package manager) appears on the home screen.

**Notes:**
- Semi-untethered: re-run Chimera after every reboot.
- Uses Sileo instead of Cydia as the package manager.
- Supports iOS 12.5.8, the latest firmware available for the iPhone 5s.

## Back Up and Restore the Activation Ticket

Once jailbroken (regardless of which method above), install [Filza File Manager](http://tigisoftware.com/cydia/) from Sileo or Zebra. The backup and restore procedure below follows [Lynnrin's activation-ticket guide](https://blog.lynnrin.moe/posts/iPhone_Unlock/). The activation ticket lives at:

```
/var/wireless/Library/Preferences/com.apple.commcenter.device_specific_nobackup.plist
```

Open the file in Filza and expand: `Root` → `kPostponementTicket` → `ActivationTicket`

### Backup

Copy the entire `ActivationTicket` value (a long Base64-encoded string) and save it somewhere safe: email, cloud note, or a file on your computer. Keep multiple copies. This is the device's activation ticket.

### Restore

After a reset, restore, or downgrade where the device needs to re-activate:

1. Activate the iPhone **without a SIM card** (skip the SIM prompt). This prevents the activation server from overwriting the ticket slot with a rejection.
2. Jailbreak again, install Filza, navigate to the same plist file.
3. Delete the current `ActivationTicket` value and paste your backed-up string.
4. Save and **reboot**.

Insert any compatible SIM card. The device should recognize it as unlocked.

## Bonus: iPhone 5s Can Downgrade to iOS 10.3.3

One of the most interesting quirks of the iPhone 5s: Apple's OTA signing servers still sign iOS 10.3.3 for A7 devices (iPhone 5s, iPad Air 1, iPad mini 2). You can downgrade from iOS 12 back to iOS 10.3.3 without saved SHSH blobs.

### Why This Works

Apple's OTA (Over-The-Air) signing mechanism is separate from the IPSW signing system. The iOS 10.3.3 IPSW has been unsigned for years, but the OTA update path remains signed for A7 devices. Whether this is intentional or simply an oversight, the community has been exploiting it for years and Apple has never revoked it.

### How to Downgrade

[Legacy iOS Kit](https://github.com/LukeZGD/Legacy-iOS-Kit) (formerly iOS-OTA-Downgrader) by LukeZGD automates the entire process:

**Requirements:**
- macOS 10.11+ or Linux (Ubuntu 22.04+, Fedora 40+, etc.)
- USB cable

**Process:**

```bash
git clone https://github.com/LukeZGD/Legacy-iOS-Kit
cd Legacy-iOS-Kit
./restore.sh
```

The tool will:
1. Put the device into DFU mode (guided)
2. Execute the checkm8 bootrom exploit via gaster
3. Request OTA signing tickets from Apple's servers (still valid)
4. Flash iOS 10.3.3 using idevicerestore

The downgrade is untethered. Once flashed, iOS 10.3.3 boots normally without any tether. All data is erased.

### Why This Matters for ICCID Unlock

This creates a nice workflow:
1. Back up the activation ticket on whatever iOS version you're currently running
2. Downgrade to iOS 10.3.3 for better performance on aging hardware
3. Jailbreak with TNS (browser-based, no PC needed for re-jailbreaks)
4. Restore the activation ticket
5. Enjoy an unlocked iPhone 5s on iOS 10.3.3

iOS 10.3.3 also runs 32-bit apps and games, [support that Apple removed in iOS 11](https://developer.apple.com/news/?id=06282017b). Together with its lighter footprint on the iPhone 5s, that makes this combination especially useful as a retro iOS gaming device.
