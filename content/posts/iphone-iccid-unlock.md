+++
title = "iPhone ICCID Unlock: Activation Ticket Backup and Restore"
date = 2026-06-22T00:00:00+00:00

[taxonomies]
categories = ["Device"]
tags = ["devices", "iOS", "carrier", "security"]
+++

ICCID activation was once a common workaround for carrier-locked iPhones. Apple patched it, so it cannot issue new unlocked tickets, but tickets created before the patch still work. To preserve an existing unlock through a reset, downgrade, or DFU restore, back up the activation ticket first and restore it afterward. The ticket is tied to one device and cannot be transferred. I use my iPhone 5s (A1453, au/KDDI Japan) as the example.

<!--more-->

## How ICCID unlock worked

Carrier-locked iPhones reject SIMs from other carriers at the activation-server level. Insert a foreign SIM and the device contacts Apple's activation server, which refuses to issue an unlocked activation record.

The ICCID method exploited the SIM's identifier rather than the phone. An ICCID (Integrated Circuit Card Identifier) is the 19 or 20 digit number that uniquely identifies a SIM card to a cellular network. It is stored on the SIM, not on the phone. A SIM interposer sat between the SIM and the phone and presented a known-good ICCID during activation. The activation server accepted that ICCID and issued an activation ticket recording an unlocked state. That ticket was then stored locally on the device.

The ticket is Apple's signed activation record. It binds the device's unique identifiers to a carrier-lock policy. Because Apple signed it, the device trusts the local copy on subsequent boots without re-querying the server. Guides sometimes call this value an "unlock certificate," but activation ticket is the precise term. It lives as a Base64 blob inside the plist detailed below.

## What survives a restore

Apple closed the ICCID path, so an interposer can no longer produce a new unlocked ticket. Existing tickets were not revoked. A device activated before the patch remains unlocked while its ticket blob remains in the plist.

Routine maintenance can clear the plist and force re-activation:

- Factory reset (Erase All Content and Settings)
- iOS upgrade or downgrade
- DFU restore

On re-activation, the device contacts Apple's server again. With the ICCID path closed, the server issues a ticket matching the phone's true lock state. For a locked phone, that is a locked ticket, overwriting the slot and losing the unlock permanently. Back up before you touch the system.

## The device

iPhone 5s model A1453, the au/KDDI Japan carrier-locked variant. It supports CDMA and GSM with LTE Band 18 and 26, au's primary bands. Mine was previously ICCID-unlocked and now runs a non-au SIM.

The 5s works well for this procedure:

- The A7 chip is vulnerable to the checkm8 bootrom exploit, so it is jailbreakable on any iOS version, permanently.
- Apple still signs iOS 10.3.3 OTA for A7 devices, allowing downgrade without saved SHSH blobs.
- It supports iOS 7 through [12.5.8](https://support.apple.com/en-us/100100), giving a wide range of jailbreak options.

## Jailbreak

Filesystem access is required to read and write the activation-ticket plist. Two reliable methods cover the versions below.

### iOS 10.3.3: TotallyNotSpyware

[TotallyNotSpyware](https://lukezgd.github.io/tns) (TNS) is a browser-based jailbreak for all 64-bit devices on iOS 10.0 to 10.3.3. It uses the SockPort exploit to reach kernel code execution from Safari, with no computer needed.

1. Open Safari on the iPhone and navigate to `https://lukezgd.github.io/tns`.
2. Slide the "Slide for Spyware" slider from left to right.
3. When the prompt appears, press "noot noot." Wait for the jailbreak to finish.
   - If the device reboots without showing the prompt, try again. The exploit does not always succeed on the first attempt.
4. Zebra, the package manager, appears on the home screen. You are jailbroken.

It is semi-untethered: the jailbreak is lost after reboot, and you revisit the TNS page to re-jailbreak. Add the page to the home screen via Share for quick access.

### iOS 12.0 to 12.5.8: Chimera

[Chimera](https://chimera.coolstar.org) is a semi-untethered jailbreak by Coolstar and the Electra Team. Its official site labels the latest download as supporting iOS 12.0 to 12.5.7, but [ios.cfw.guide](https://ios.cfw.guide/installing-chimera/) confirms Chimera works on A7 through A11 devices through iOS 12.5.8. For an iPhone 5s on iOS 12.4.1 to 12.5.8, the guide recommends an unofficial build with [chimera_patch](https://github.com/staturnzz/chimera_patch) for better success rates. The original build also works.

1. Sideload the Chimera IPA using AltStore, Sideloadly, or TrollStore if available on your iOS version.
2. Open the Chimera app and tap "Jailbreak."
3. Wait for the respring. Sileo, the package manager, appears on the home screen.

It is semi-untethered: re-run Chimera after every reboot. It uses Sileo instead of Cydia and supports iOS 12.5.8, the last firmware available for the iPhone 5s.

## Back up and restore the ticket

Once jailbroken, install [Filza File Manager](http://tigisoftware.com/cydia/) from Sileo or Zebra. The procedure below follows [Lynnrin's activation-ticket guide](https://blog.lynnrin.moe/posts/iPhone_Unlock/). The ticket lives at:

```
/var/wireless/Library/Preferences/com.apple.commcenter.device_specific_nobackup.plist
```

Open the file in Filza and expand `Root` → `kPostponementTicket` → `ActivationTicket`.

### Backup

Copy the entire `ActivationTicket` value, a long Base64 string, to durable storage such as email, a cloud note, or a file on your computer. Keep several copies. It is the only value that preserves the unlock.

### Restore

After any reset, restore, or downgrade that forces re-activation:

1. Activate the iPhone **without a SIM card**, skipping the SIM prompt. Activating with a SIM in place lets the server write a fresh locked ticket into the slot before you can intervene.
2. Jailbreak again, install Filza, and open the same plist.
3. Delete the current `ActivationTicket` value and paste your backed-up string.
4. Save and **reboot**.

Insert any compatible SIM. The device reads the restored ticket and treats itself as unlocked.

## Downgrade to iOS 10.3.3

Apple's OTA signing servers still sign iOS 10.3.3 for A7 devices (iPhone 5s, iPad Air 1, iPad mini 2). You can downgrade from iOS 12 to iOS 10.3.3 without saved SHSH blobs.

### Why this works

Apple's OTA signing path is separate from the IPSW signing system. The iOS 10.3.3 IPSW has been unsigned for years, but the OTA update path remains signed for A7 devices. Whether this is intentional or an oversight, the community has used it for years and Apple has never revoked it.

### How to downgrade

[Legacy iOS Kit](https://github.com/LukeZGD/Legacy-iOS-Kit) (formerly iOS-OTA-Downgrader) by LukeZGD automates the process.

Requirements: macOS 10.11 or later, or Linux (Ubuntu 22.04+, Fedora 40+, etc.), and a USB cable.

```bash
git clone https://github.com/LukeZGD/Legacy-iOS-Kit
cd Legacy-iOS-Kit
./restore.sh
```

The tool puts the device into DFU mode, with prompts to guide you through it. It runs the checkm8 bootrom exploit through gaster, requests OTA signing tickets from Apple's servers, and flashes iOS 10.3.3 with idevicerestore. The restored system boots normally afterward without a tether. All data is erased.

### Restoring the unlock after a downgrade

1. Back up the activation ticket on whatever iOS version you currently run.
2. Downgrade to iOS 10.3.3 for better performance on aging hardware.
3. Jailbreak with TNS, which is browser-based and needs no PC for re-jailbreaks.
4. Restore the activation ticket.

iOS 10.3.3 also runs 32-bit apps and games, [support that Apple removed in iOS 11](https://developer.apple.com/news/?id=06282017b). With its lighter footprint on the iPhone 5s, that makes this combination useful as a retro iOS gaming device.
