+++
title = "Rescuing a Bricked 360T7 Router via UART After a DDR3 RAM Upgrade"
date = 2026-05-23T00:00:00+00:00

[taxonomies]
categories = ["Device"]
tags = ["firmware", "devices", "networking"]
+++

A memory-modded 360T7 ran fine for years until I flashed OpenWrt's bl2 and fip, which bricked it with `dram size: 0MB`. Every rescue package found online failed. Eventually I traced the issue to a DDR3 frequency mismatch in the ATF source, compiled a custom bl2, and recovered the router via UART.

<!--more-->

## Background

A few years back I picked up a modded 360T7 on Xianyu (a Chinese secondhand marketplace) where the seller had replaced the original 256MB DDR3 chip with a 512MB one. Flashed hanwckf's U-Boot and Lean's QWRT, ran rock solid for years without a single issue.

Recently I had time off and wanted to switch the entire stack to upstream OpenWrt for the latest features. Following the official docs, I SSH'd into the router and flashed OpenWrt's bl2 and fip, erased the firmware partition, and the router never came back. Couldn't enter recovery to pull TFTP firmware. Connected TTL and saw serial output stuck at `EMI detected dram size: 0MB` - memory initialization failed, boot chain broken at the bl2 stage.

## Investigation

Searched everywhere online, found that MediaTek platforms have a UART rescue mode (mtk_uartboot) that can push bl2 to the SoC's SRAM via serial for temporary boot. Tried every 360T7 rescue bl2 and fip shared by the community, all failed with the same `dram size: 0MB`.

First thought was the 512MB memory chip had died. But I spotted someone on Xianyu selling remote repair services for 360T7 with identical symptoms. If others can fix it with software, it's not a hardware problem. Besides, this thing ran fine for years.

Had time on my hands, decided to sort it out from source. Left U-Boot alone, kept using the bl-mt798x repo that had always worked, minimized variables and focused on DDR configuration.

Opened up the router to confirm the chip: Zentel A3T4GF40ABF (DDR3-1600). Checked 360T7's default defconfig - no DDR3 frequency specified, and MT7981's Kconfig defaults to DDR3-2133:

```
# plat/mediatek/apsoc_common/Config.in
config _PLAT_MT7981
    select _DEFAULT_DDR3_2133
```

Found the problem: the DDR3-1600 chip was being driven at 2133MT/s, DRAM calibration failed outright. The previous hanwckf firmware worked because the seller had already set the correct frequency when flashing. OpenWrt's official bl2 and community rescue packages were all compiled for DDR3-2133, naturally incompatible.

Worth noting that this was later fixed upstream in OpenWrt - [PR #22797](https://github.com/openwrt/openwrt/pull/22797) merged on 2026-05-09, changing 360T7's BL2 from DDR3-2133 to DDR3-1866 and adding UBI layout support. But I had flashed the latest OpenWrt stable at the time, which didn't include this fix.

MT7981 ATF only supports two DDR3 frequency options: `_DDR3_FREQ_1866` (1866MT/s) and `_DDR3_FREQ_2133` (2133MT/s, default). No 1600 option, lowest available is 1866, which is about 17% overclock for this chip. DDR3 stepping up one bin is usually fine.

After identifying the frequency issue, I compiled a DDR3-1866 bl2 and tried pushing it via mtk_uartboot. Timeout, handshake failed. Another trap: the default build uses the SPI-NAND boot path and doesn't include UART download handshake code. Confirmed in source that bl2 needs both `_BOOT_DEVICE_RAM=y` and `_RAM_BOOT_RAM_BOOT_UART_DL=y` to work with mtk_uartboot - the former selects the RAM boot path compiling `bl2_boot_ram.c`, the latter enables UART handshake compiling `uart_dl.c`.

## Solution

Two things to compile: a UART rescue bl2 (RAM boot + UART_DL + DDR3-1866, temporary boot only), and a production bl2 + FIP for flashing to NAND (SPI-NAND boot + DDR3-1866).

### UART Boot bl2

Create `configs/mt7981_360t7_uart_defconfig`:

```
_PLAT_MT7981=y
_MT7981_BOARD_BGA=y
_DRAM_DDR3=y
_DDR3_FREQ_1866=y
_LOG_LEVEL_INFO=y
_BOOT_DEVICE_RAM=y
_RAM_BOOT_RAM_BOOT_UART_DL=y
```

Build (can't use the repo's `build.sh` - it hardcodes defconfig names and builds FIP):

```bash
rm -rf atf-20240117-bacca82a8/build
make -C atf-20240117-bacca82a8 mt7981_360t7_uart_defconfig \
    CONFIG_CROSS_COMPILER=aarch64-linux-gnu- CROSS_COMPILER=aarch64-linux-gnu-
make -C atf-20240117-bacca82a8 bl2 \
    CONFIG_CROSS_COMPILER=aarch64-linux-gnu- CROSS_COMPILER=aarch64-linux-gnu- -j$(nproc)
```

Output: `build/mt7981/release/bl2.bin`, renamed to `uart_bl2.bin`.

### Production bl2 + FIP

Modify `configs/mt7981_360t7_defconfig`:

```
_PLAT_MT7981=y
_MT7981_BOARD_BGA=y
_DRAM_DDR3=y
_DDR3_FREQ_1866=y
_LOG_LEVEL_INFO=y
```

Build:

```bash
SOC=mt7981 BOARD=360t7 ./build.sh
```

Output: `output/mt7981_360t7-bl2.bin` (flash_bl2.bin) and `output/mt7981_360t7-fip-fixed-parts.bin` (flash_fip.bin).

## Rescue Procedure

Boot via UART into U-Boot:

```bash
mtk_uartboot -s /dev/ttyUSB0 -p uart_bl2.bin --aarch64 -f flash_fip.bin
```

Serial output shows `EMI: detected dram size: 512MB`, successfully enters `MT7981>` U-Boot command line.

Flash via TFTP (PC connected to router's LAN port, U-Boot IP is `192.168.1.1`, PC set to `192.168.1.2/24`, TFTP server running):

```
tftpboot 0x46000000 flash_bl2.bin
mtd erase bl2
mtd write bl2 0x46000000 0x0 $filesize

tftpboot 0x46000000 flash_fip.bin
mtd erase fip
mtd write fip 0x46000000 0x0 $filesize

reset
```

Rebooted, flashed QWRT back, booted successfully, 512MB memory fully recognized, all good.

## Rescue Files Download

I've uploaded the compiled files. Since the DDR3 frequency is lowered to 1866MT/s, these should have a higher success rate for memory-modded 360T7 units:

[https://drive.qzydustin.com/360t7](https://drive.qzydustin.com/360t7)

Contains `uart_bl2.bin`, `flash_bl2.bin`, `flash_fip.bin`. Follow the steps above.

## Reference

### MTD Partition Layout

Stock firmware (dual firmware partitions):

```
mtdparts=nmbm_spim_nand:1024k(bl2),512k(u-boot-env),2048k(Factory),2048k(fip),36M(ubi),36M(firmware-1),36M(plugin),1M(config),512k(factory),7M(log)
```

| Partition | Size | Description |
|-----------|------|-------------|
| bl2 | 1024K | BL2 bootloader |
| u-boot-env | 512K | U-Boot environment variables |
| Factory | 2048K | Factory calibration data |
| fip | 2048K | FIP (bl31 + U-Boot) |
| ubi | 36M | Primary firmware |
| firmware-1 | 36M | Backup firmware |
| plugin | 36M | Plugin partition |
| config | 1M | Configuration |
| factory | 512K | Factory info |
| log | 7M | Log |

hanwckf bl-mt798x partition layout:

```
mtdparts=nmbm_spim_nand:1024k(bl2),512k(u-boot-env),2048k(Factory),2048k(fip),108M(ubi),1M(config),512k(factory),7M(log)
```

| Partition | Size | Description |
|-----------|------|-------------|
| bl2 | 1024K | BL2 bootloader |
| u-boot-env | 512K | U-Boot environment variables |
| Factory | 2048K | Factory calibration data |
| fip | 2048K | FIP (bl31 + U-Boot) |
| ubi | 108M | System firmware (merged stock ubi + firmware-1 + plugin) |
| config | 1M | Configuration |
| factory | 512K | Factory info |
| log | 7M | Log |

Pitfall log:

| # | Action | Result | Cause |
|---|--------|--------|-------|
| 1 | Flashed official OpenWrt bl2/fip per docs | Bricked, `dram size: 0MB` | Official firmware compiled for DDR3-2133, incompatible with 1600 chip |
| 2 | Community rescue bl2 | Same 0MB | Also 2133 frequency |
| 3 | Self-compiled DDR3-1866 bl2 + mtk_uartboot | Timeout | SPI-NAND boot mode, no UART_DL code |
| 4 | UART bl2 + DDR3-1866 + RAM boot | Success | Frequency and boot mode both correct |

### Links

- [mtk_uartboot](https://github.com/981213/mtk_uartboot) - MediaTek SoC UART rescue tool
- [MediaTek Filogic Serial Rescue Guide](https://www.cnblogs.com/p123/p/18046679) - MT7981/MT7986 detailed walkthrough
- [bl-mt798x Source](https://github.com/hanwckf/bl-mt798x) - ATF + U-Boot build
- [OpenWrt Wiki - Qihoo 360T7](https://openwrt.org/toh/qihoo/360t7_1.0) - Official device page
- [OpenWrt PR #22797](https://github.com/openwrt/openwrt/pull/22797) - DDR3 frequency fix + UBI layout

## Addendum: NMBM Compatibility on 360T7

While this rescue was about DDR frequency, the investigation also revealed another major pitfall in the 360T7 ecosystem: NMBM (NAND Mapping Block Management) compatibility.

NMBM is MediaTek's proprietary NAND bad block management layer. When enabled, it reserves space at the end of flash for block mapping and must be enabled/disabled consistently in both ATF (BL2) and Linux kernel - mismatch causes problems. Critical data on NAND (Factory calibration, fip) without bad block protection can be lost if preceding blocks go bad, shifting fixed offsets. Different firmware takes different approaches:

- **NMBM** (MediaTek proprietary, used by hanwckf): builds a bad block mapping table for the entire NAND at the BL2 stage. Upper layers see a "bad-block-free" virtual device, all partitions including Factory calibration are protected
- **OpenWrt default layout** (old approach): uses a fixed-partitions framework overall. Only the ubi partition holding rootfs/kernel has UBI bad block management. Other partitions (Factory, fip) sit at raw fixed offsets on NAND with no protection
- **OpenWrt UBI layout** (new approach, PR #22797): no NMBM, but puts Factory and fip inside UBI volumes so all critical data is under bad block management

| Firmware/U-Boot | NMBM | Factory Protection | Partition Scheme |
|---|---|---|---|
| 360 stock | Enabled | NMBM transparent mapping | Dual firmware (ubi 36M + firmware-1 36M) |
| hanwckf bl-mt798x | Enabled | NMBM transparent mapping | Large ubi 108M + trailing partitions |
| OpenWrt default | Disabled | None (fixed-partitions, bad block risk) | Large ubi 108M + trailing partitions |
| OpenWrt UBI (PR #22797, snapshot only) | Disabled | UBI volume management | Only bl2 1M fixed, rest in UBI volumes (127M) |

The key issue: **OpenWrt firmware is incompatible with hanwckf U-Boot**. hanwckf's ATF enables NMBM while OpenWrt's ATF does not. NMBM reserves space at the end of NAND for mapping, so the two see different usable partition sizes. Mixing them causes UBI volume table checksum failures and potentially another brick. Bottom line: **flash the full stack together** - bl2 + fip + firmware must all agree on NMBM and partition scheme, no mixing.

### Practical Advice

If your 360T7 runs hanwckf U-Boot, don't flash OpenWrt UBI layout firmware alone. Either stick with immortalwrt-mt798x / QWRT and compatible firmware, or switch the entire stack to OpenWrt's official BL2 + FIP + firmware (but watch out for the DDR frequency issue discussed in this article).

Good news: PR #22797 (merged 2026-05-09) fixes both DDR3 frequency and adds UBI layout. Once a future OpenWrt stable release ships with this, memory-modded 360T7 units should no longer hit the frequency problem described here.

## Addendum: 360T7 Firmware Comparison

Since we've touched on firmware compatibility, here's an overview of available firmware options for the 360T7.

| | OpenWrt Official | ImmortalWrt Official | hanwckf | QWRT (Lean) |
|---|---|---|---|---|
| Base | OpenWrt mainline | ImmortalWrt (OpenWrt fork) | ImmortalWrt 21.02 fork | Lean's LEDE |
| License | Open source | Open source | Open source (WiFi blob closed) | Closed source, paid |
| Distribution | Official site | Official site | GitHub source / community builds | QQ group, paid binary |
| Paired U-Boot | OpenWrt official U-Boot | ImmortalWrt official U-Boot | hanwckf bl-mt798x | hanwckf bl-mt798x |
| Kernel | 6.6 (24.10) | 6.6 (24.10) | 5.4 (locked to 21.02) | Undisclosed |
| WiFi Driver | Open source mt76 | Open source mt76 | MTK closed source | MTK closed source |
| Hardware Offload | Hardware flow offloading | Hardware flow offloading | MTK SDK native HNAT | MTK SDK native HNAT |

### Key Differences

**Open source mt76 vs MTK closed source driver**: the closed source driver has advantages in signal strength, multi-client concurrency, and roaming. Open source mt76 is still catching up.

**OpenWrt vs ImmortalWrt**: ImmortalWrt carries patches not accepted upstream and China-region packages, with more aggressive kernel configuration.

**hanwckf stuck on 21.02**: the MTK closed source driver is tied to the old kernel SDK and cannot upgrade to newer kernels.

**Hardware offloading**: mainline firmware uses Hardware flow offloading which supports MT7981, though some users report suboptimal latency under heavy load. hanwckf/QWRT use MTK SDK native HNAT, closer to the stock implementation.
