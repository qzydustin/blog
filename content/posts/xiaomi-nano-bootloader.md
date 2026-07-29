+++
title = "Flashing the Xiaomi Router Nano Bootloader"
date = 2022-09-28T14:43:27-07:00

[taxonomies]
categories = ["Device"]
tags = ["firmware", "bootloader", "networking"]
+++

The Xiaomi Router Nano is a MediaTek MT7628 router. Stock MiWiFi accepts only signed updates and exposes no shell. Replacing the bootloader lets it run OpenWrt or Padavan.

<!--more-->

## What a bootloader does

A bootloader is the first code the CPU runs after reset. It initializes RAM and flash, locates the firmware, and jumps into it. On this class of MIPS router it occupies its own flash partition, separate from the firmware. The two can be reflashed independently. U-Boot is the open-source bootloader most embedded Linux boards use, and the stock bootloader on these routers is derived from it.

## Recovery versus brick risk

Breed runs before the firmware and is reachable with the reset button, so it can recover from a bad firmware flash. The bootloader itself is riskier to replace. If the wrong image is written to the Bootloader partition, there is no software recovery path and you need an external SPI programmer for the flash chip.

## Gaining root first

You cannot flash the bootloader through the stock web UI. You need a root shell, which on MiWiFi firmware means enabling SSH. A root shell gives you the `mtd` tool, the only way to write the Bootloader partition on this firmware.

Two conditions must hold. The firmware must be the development build, downloadable from [miwifi.com](https://www1.miwifi.com/miwifi_download.html). The stable build ships without SSH support compiled in. The `ssh_en` nvram flag must also be set. The Nano has no USB port, so its default `ssh_en` is already `1`.

With SSH enabled, the root password is derived from the router's serial number by a fixed algorithm. Anyone who can read the S/N sticker can compute it.

```python
import sys
import hashlib

# Xiaomi router r1d is different from others
salt = {'r1d': 'A2E371B0-B34B-48A5-8C40-A7133F3B5D88',
        'others': 'd44fb0960aa0-a5e6-4a30-250f-6d2df50a'}


def get_salt(sn):
    if "/" not in sn:
        return salt["r1d"]

    return "-".join(reversed(salt["others"].split("-")))


def calc_passwd(sn):
    passwd = sn + get_salt(sn)
    m = hashlib.md5(passwd.encode())
    return m.hexdigest()[:8]


if __name__ == "__main__":
    if len(sys.argv) != 2:
        print(f"Usage: {sys.argv[0]} <S/N>")
        sys.exit(1)

    serial = sys.argv[1]
    print(calc_passwd(serial))
```

## Backing up before you touch the bootloader

Back up the full flash before writing anything. This dump is your last-resort recovery material. Keep a copy off the router.

```shell
dd if=/dev/mtd0 of=/tmp/all.bin
```

Copy it off the router with SCP. If you brick the board later and own a SPI programmer, this is the image you write back to the chip.

## Flashing Breed

This guide installs [Breed](https://breed.hackpascal.net). Download [breed-mt7688-reset38.bin](https://breed.hackpascal.net/breed-mt7688-reset38.bin) and copy it to `/tmp` on the router.

```shell
mtd -r write /tmp/breed-mt7688-reset38.bin Bootloader
```

The `-r` reboots the router when the write finishes. Do not interrupt power during the write.

## Entering Breed

To reach the Breed web UI after flashing:

1. Plug a cable into a LAN port, not the blue WAN port.
2. Disconnect power, hold Reset, reconnect power, and keep holding Reset until the yellow LED starts blinking (about 5 seconds).
3. Open `192.168.1.1` in a browser.

From here you flash firmware through Breed's web UI instead of the stock firmware's signed-update channel.
