+++
title = "Dual-Partition Sysupgrade on the 360V6 with OpenWrt"
date = 2026-05-23T00:00:00+00:00

[taxonomies]
categories = ["Device"]
tags = ["openwrt", "firmware", "networking"]
+++

The 360V6 has A/B dual-partition flash: two complete rootfs copies and a bootloader slot flag that selects one at boot. Updates write the inactive partition and leave the running one alone. If a write fails or is interrupted, the router can still boot the old system.

OpenWrt's sysupgrade ignored the layout and wrote the running partition in place. A power loss or bad image mid-write left the device unbootable, recoverable only over TTL. I added proper A/B support upstream, and it has merged.

<!--more-->

## Background

The 360V6 runs a Qualcomm IPQ6000 SoC with NAND flash. The factory firmware uses an A/B dual-partition layout: two rootfs partitions, slot0 and slot1, with a slot flag in the bootconfig partition selecting the active one. At boot the bootloader reads that flag and remaps the selected physical partition as `mtd16` (`rootfs`), the other as `mtd17` (`rootfs_1`). The mapping is dynamic and transparent to Linux. At runtime `mtd16` is always the running system and `mtd17` is always the alternate, regardless of which physical slot they are.

Flashing OpenWrt through initramfs creates another issue. It writes OpenWrt to only one partition, leaving OEM UBI volumes (`wifi_fw`, `ubi_rootfs`) on the other. Those volumes block later sysupgrade writes to the alternate partition and [cause UBI operation failures](https://github.com/openwrt/openwrt/issues/19062).

## bootconfig structure

The slot flag lives in the bootconfig partition (`mtd2` and `mtd3`, redundant copies). Binary layout:

| Offset | Size | Description |
|--------|------|-------------|
| 0x00 | 4 | Magic: `a0 a1 a2 a3` |
| 0x04 | 4 | Header version / length |
| 0x08 | 4 | Entry count / flags |
| 0x0C | N×20 | Partition entries (16-byte name + 4-byte primaryboot flag) |
| 0x94 | 1 | rootfs entry's primaryboot field |
| 0x14C | 4 | End magic: `b0 b1 b2 b3` |

The `rootfs` entry's primaryboot field at offset `0x94` selects the boot slot: `0x00` for slot0 (NAND offset 18350080), `0x01` for slot1 (NAND offset 65142784).

## Solution

The change has two parts. `bootconfig.sh` provides shell functions that validate the magic, find a partition entry by name, and read or toggle the primaryboot flag. The 360V6 upgrade flow in `platform.sh` calls them:

```sh
qihoo,360v6)
    CI_UBIPART="rootfs_1"
    qihoo_bootconfig_toggle_rootfs "0:bootconfig"
    remove_oem_ubi_volume wifi_fw
    remove_oem_ubi_volume ubi_rootfs
    nand_do_upgrade "$1"
    ;;
```

`CI_UBIPART="rootfs_1"` sends `nand_do_upgrade` to the alternate partition. `qihoo_bootconfig_toggle_rootfs` reads the first 336 bytes of bootconfig into a temporary file, flips the rootfs primaryboot value, and writes the result to both bootconfig partitions. `remove_oem_ubi_volume` removes the OEM volumes from the alternate partition before the write.

After I submitted the change, the maintainer pointed out similar bootconfig handling for the Alfa Network AP120C-AX and asked me to reuse it. I refactored it around that existing framework. Sysupgrade now writes the alternate partition, toggles the slot, and reboots.

## References

- [OpenWrt PR #21154](https://github.com/openwrt/openwrt/pull/21154) - The PR discussed in this post
- [OpenWrt Issue #19062](https://github.com/openwrt/openwrt/issues/19062) - OEM volumes causing sysupgrade failure
- [OpenWrt PR #15940](https://github.com/openwrt/openwrt/pull/15940) - 360V6 initramfs installation support

## Appendix: 360V6 Partition Table and Flashing Guide

### NAND Partition Layout

```
dev:    size   erasesize  name
mtd0: 00180000 00020000 "0:sbl1"
mtd1: 00100000 00020000 "0:mibib"
mtd2: 00080000 00020000 "0:bootconfig"
mtd3: 00080000 00020000 "0:bootconfig1"
mtd4: 00380000 00020000 "0:qsee"
mtd5: 00380000 00020000 "0:qsee_1"
mtd6: 00080000 00020000 "0:devcfg"
mtd7: 00080000 00020000 "0:devcfg_1"
mtd8: 00080000 00020000 "0:rpm"
mtd9: 00080000 00020000 "0:rpm_1"
mtd10: 00080000 00020000 "0:cdt"
mtd11: 00080000 00020000 "0:cdt_1"
mtd12: 00080000 00020000 "0:appsblenv"
mtd13: 00180000 00020000 "0:appsbl"
mtd14: 00180000 00020000 "0:appsbl_1"
mtd15: 00080000 00020000 "0:art"
mtd16: 02ca0000 00020000 "rootfs"
mtd17: 02ca0000 00020000 "rootfs_1"
mtd18: 00080000 00020000 "0:ethphyfw"
mtd19: 00800000 00020000 "log"
mtd20: 00c00000 00020000 "plugin"
mtd21: 00080000 00020000 "config"
mtd22: 00040000 00020000 "factory"
```

Key partitions:

| Partition | Description |
|-----------|-------------|
| mibib | Partition table, defines the entire NAND layout |
| bootconfig / bootconfig1 | Qualcomm SMEM boot configuration, stores slot flag |
| appsbl / appsbl_1 | Qualcomm custom U-Boot |
| rootfs / rootfs_1 | A/B system partitions, each containing a full UBI container |
| art | RF calibration data, stores WiFi MAC addresses and calibration parameters |

Paid third-party mibib and appsbl images online merge the dual rootfs into one large partition for more firmware space. Do not use them. They destroy the factory A/B layout. The 360V6 has a USB port for storage expansion, and the stock partition sizes are enough for daily use.

### Stock Firmware Upgrade Package Structure and Extraction

The stock upgrade package is `[ Upgrade ELF ] + [ DTB ] + [ UBI Payload ]`. The Upgrade ELF is an ARM32 static executable that drives the upgrade flow. The UBI Payload is a complete rootfs slot image: kernel, ubi_rootfs, rootfs_data, wifi_fw.

The stock updater also writes the non-current slot, sets the slot flag, and reboots.

Extracting the UBI Payload from the upgrade package:

```bash
binwalk 360V6-v4.2.13.3033-rel-upgrade.bin
# Output: UBI image, version: 1, image size: 27918336 bytes, offset: 0x92400

dd if=360V6-v4.2.13.3033-rel-upgrade.bin \
   of=upg_payload.ubi \
   bs=1 skip=599040 count=27918336
```

### Flashing Methods

**Method 1: Flash via Telnet on stock firmware**

Prerequisites: stock firmware 4.2.13.3033, and the `360_v6_jd_telnetd_ctrl_app_nda_signed.opk` package installed to enable Telnet (root / telnetdebug, port 23).

```bash
ubiformat /dev/mtd17 -y -f openwrt-qualcommax-ipq60xx-qihoo_360v6-squashfs-factory.ubi
sync
```

**Method 2: Flash after booting initramfs from U-Boot**

Interrupt boot at the U-Boot stage:

```bash
tftpboot initramfs.itb
bootm
```

Once OpenWrt boots, upload `sysupgrade.bin` by scp or LuCI and run sysupgrade.

**Reverting to stock firmware**

From OpenWrt, write the extracted UBI payload to the alternate partition:

```bash
ubiformat /dev/mtd17 -y -f upg_payload.ubi
sync
```

Whether flashing OpenWrt or reverting to stock, writing the alternate partition takes effect only after a slot toggle (see the bootconfig structure section above).

### TTL Recovery

In U-Boot, `flash rootfs` writes to whichever physical rootfs partition bootconfig currently selects. To flash the other one, change the slot flag first.

```bash
tftpboot 0x44000000 firmware.bin
flash rootfs 0x44000000 $filesize
```

Only flash one rootfs partition when recovering. Do not write identical UBI images to both partitions. They conflict. After TTL recovery, run sysupgrade again, because UBI bad-block markers from a TTL flash may not match the real NAND state.

### Slot Flag Verification

By switching boot slots repeatedly and watching, the physical offsets of mtd16/mtd17 swap accordingly (`cat /sys/class/mtd/mtd16/offset`), which confirms the mapping is assigned by the bootloader at runtime. Inspect the bootconfig bytes around offset 0x94:

```bash
dd if=/dev/mtd2 bs=1 skip=$((0x80)) count=64 2>/dev/null | hexdump -C
```

Change the byte at 0x94, write it back to both bootconfig partitions, reboot, and watch the offset swap.
