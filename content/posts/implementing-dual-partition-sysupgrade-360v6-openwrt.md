+++
title = "Implementing Dual-Partition Sysupgrade for the 360V6 Router on OpenWrt"
date = 2026-05-23T00:00:00+00:00

[taxonomies]
categories = ["Device"]
tags = ["firmware", "devices", "networking", "openwrt"]
+++

The 360V6 ships with an A/B dual-partition design, but after flashing OpenWrt, sysupgrade only writes to the current partition while OEM remnants on the alternate partition cause upgrade failures. After figuring out the bootconfig slot mechanism, I submitted dual-partition upgrade support to OpenWrt upstream, which has since been merged.

<!--more-->

## Background

The 360V6 is based on the Qualcomm IPQ6000 SoC with NAND storage and features a factory A/B dual-system design. Two rootfs partitions (slot0 / slot1) are controlled by a slot flag in the bootconfig partition. At boot, the bootloader reads this flag and maps the selected physical partition as mtd16 (rootfs), the other as mtd17 (rootfs_1). This mapping is entirely transparent to Linux: at runtime, mtd16 is always the current system and mtd17 is always the alternate.

After flashing OpenWrt, two problems emerged: first, sysupgrade didn't leverage the dual-partition mechanism, overwriting the current partition directly and wasting the A/B redundancy. Second, when installing via initramfs, only one partition gets OpenWrt while the alternate retains non-standard OEM UBI volumes (`wifi_fw`, `ubi_rootfs`). These volumes interfere with subsequent sysupgrade writes to the alternate partition, [causing UBI operation failures](https://github.com/openwrt/openwrt/issues/19062).

## bootconfig Structure

Boot slot control is stored in the bootconfig partition (mtd2 and mtd3, redundant copies). Binary layout:

| Offset | Size | Description |
|--------|------|-------------|
| 0x00 | 4 | Magic: `a0 a1 a2 a3` |
| 0x04 | 4 | Header version / length |
| 0x08 | 4 | Entry count / flags |
| 0x0C | N×20 | Partition entries (16-byte name + 4-byte primaryboot flag) |
| 0x94 | 1 | rootfs entry's primaryboot field |
| 0x14C | 4 | End magic: `b0 b1 b2 b3` |

The `rootfs` entry's primaryboot field (offset 0x94) determines which physical slot the bootloader boots: 0x00 for slot0 (NAND offset 18350080), 0x01 for slot1 (NAND offset 65142784).

## Solution

The implementation has two parts. First, a new `bootconfig.sh` providing shell functions to parse the bootconfig structure: validate magic, look up partition entry index by name, read and toggle the primaryboot flag. Second, the upgrade flow for 360V6 in `platform.sh`:

```sh
qihoo,360v6)
    CI_UBIPART="rootfs_1"
    qihoo_bootconfig_toggle_rootfs "0:bootconfig"
    remove_oem_ubi_volume wifi_fw
    remove_oem_ubi_volume ubi_rootfs
    nand_do_upgrade "$1"
    ;;
```

`qihoo_bootconfig_toggle_rootfs` reads the first 336 bytes of bootconfig into a temp file, toggles the rootfs primaryboot value, and writes it back to both bootconfig partitions. `remove_oem_ubi_volume` cleans up OEM remnant volumes on the alternate partition so `nand_do_upgrade` can write cleanly.

After submission, the maintainer pointed out that ipq60xx already had similar bootconfig handling for the Alfa Network AP120C-AX and suggested reusing it. Refactored to build on the existing framework, squashed and merged. The end result is sysupgrade becoming a true A/B upgrade: write to the alternate partition, toggle slot, reboot. The current system is untouched during upgrade, and the old system remains bootable if anything goes wrong.

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

There are paid third-party mibib and appsbl images available online that merge the dual rootfs into a single large partition for more firmware space. This is not recommended: it destroys the factory dual-partition structure and loses A/B redundancy. The 360V6 has a USB port for storage expansion if needed, and the stock partition sizes are sufficient for daily use.

### Stock Firmware Upgrade Package Structure and Extraction

The stock upgrade package consists of three parts: `[ Upgrade ELF ] + [ DTB ] + [ UBI Payload ]`

- **Upgrade ELF**: ARM32 static executable handling the upgrade flow
- **UBI Payload**: Complete rootfs slot image (kernel / ubi_rootfs / rootfs_data / wifi_fw)

The stock upgrade flow follows the same logic as this implementation: select the non-current slot, write UBI payload, update slot flag, reboot.

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

Prerequisites: upgrade to firmware 4.2.13.3033, install `360_v6_jd_telnetd_ctrl_app_nda_signed.opk` to enable Telnet (root / telnetdebug, port 23).

```bash
ubiformat /dev/mtd17 -y -f openwrt-qualcommax-ipq60xx-qihoo_360v6-squashfs-factory.ubi
sync
```

**Method 2: Flash after booting initramfs from U-Boot**

Interrupt boot at U-Boot stage:

```bash
tftpboot initramfs.itb
bootm
```

Once OpenWrt boots, upload `sysupgrade.bin` via scp or LuCI to upgrade.

**Reverting to stock firmware**

From OpenWrt, write the extracted UBI payload to the alternate partition:

```bash
ubiformat /dev/mtd17 -y -f upg_payload.ubi
sync
```

Whether flashing OpenWrt or reverting to stock, writing to the alternate partition requires a slot toggle to take effect (see the bootconfig structure section above).

### TTL Recovery

In U-Boot, the `flash rootfs` command writes to whichever physical rootfs partition is currently selected by bootconfig. To flash the other physical partition, modify the bootconfig slot flag first.

```bash
tftpboot 0x44000000 firmware.bin
flash rootfs 0x44000000 $filesize
```

Note: only flash one rootfs partition when recovering. Do not flash identical UBI images to both partitions simultaneously as they will conflict. After TTL recovery, it's recommended to run sysupgrade again, since the UBI bad block markers from a TTL flash may not match the actual NAND state.

### Slot Flag Verification

By switching boot slots multiple times and observing, the physical offsets of mtd16/mtd17 swap accordingly (`cat /sys/class/mtd/mtd16/offset`), confirming the mapping is dynamically assigned by the bootloader. Inspect the bootconfig contents around offset 0x94:

```bash
dd if=/dev/mtd2 bs=1 skip=$((0x80)) count=64 2>/dev/null | hexdump -C
```

Modify the byte at 0x94, write back to both bootconfig partitions, reboot, and observe the offset swap to verify.
