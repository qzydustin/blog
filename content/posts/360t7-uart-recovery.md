+++
title = "Recovering a Bricked 360T7 via UART After a DDR3 RAM Upgrade"
date = 2026-05-23T00:00:00+00:00

[taxonomies]
categories = ["Device"]
tags = ["firmware", "devices", "networking"]
+++

A memory-modded 360T7 had run for years until I flashed OpenWrt's bl2 and fip. It then stopped at `dram size: 0MB`. Every rescue package I found failed for the same reason: OpenWrt's official bl2 drives DDR3 at 2133MT/s, while this router has a DDR3-1600 chip. DRAM initialization failed before the rest of the boot chain could load. I traced the settings in the ATF source, built a bl2 at 1866MT/s, and recovered the router over UART.

<!--more-->

## How this router is built

A few years back I picked up a modded 360T7 on Xianyu, a Chinese secondhand marketplace. The seller had desoldered the stock 256MB DDR3 chip and fitted a 512MB one: a Zentel A3T4GF40ABF, rated DDR3-1600. They flashed hanwckf's U-Boot and Lean's QWRT, set the DDR3 frequency to match the new chip, and it ran without issue for years.

I recently wanted to move the whole stack to upstream OpenWrt. Following the official docs, I SSH'd in, flashed OpenWrt's bl2 and fip, erased the firmware partition, and the router never came back. I could not enter recovery to pull TFTP firmware. Over TTL I saw serial output frozen at `EMI detected dram size: 0MB`. Memory initialization had failed and the boot chain was broken at the bl2 stage.

## Why a RAM swap can brick the bootloader

On MediaTek MT7981, the boot chain runs BL2, then BL31, then U-Boot, then the kernel. BL2 is the ARM Trusted Firmware first-stage bootloader, and its first job is DRAM initialization. It programs the memory controller with a target frequency and a timing table, runs DRAM training, and reports the usable size. Only once DRAM is alive can BL2 load the later stages into it.

Those timings are not read from the fitted chips at runtime. They are compile-time constants in the ATF, selected by Kconfig options, and the frequency option picks a pre-baked timing table for that speed bin. If the compiled frequency does not match what the fitted DRAM can run, training fails and the controller reports zero usable bytes. Every stage after BL2 needs working external RAM, so a failed DRAM init stops the entire boot.

This router's seller had flashed hanwckf's U-Boot with the frequency set for the 512MB DDR3-1600 chip, which is why it ran for years. OpenWrt's official bl2 is compiled for DDR3-2133, the MT7981 default. A DDR3-1600 chip cannot train at 2133MT/s. Flashing OpenWrt's bl2 replaced a matched bootloader with a mismatched one, and the chain died at `dram size: 0MB`.

## Why UART is the only way in

With DRAM initialization broken, the flash bootloader cannot get past its own bad configuration. The router has no usable RAM in which to load a network image. The only path left is the SoC's boot ROM.

MediaTek boot ROMs support a UART download mode. In this mode the ROM listens on the UART pins for a host to push a small image into the SoC's internal SRAM. SRAM is on-die and needs no initialization, so this works with DRAM dead and flash untouched. The pushed image is a rescue BL2. It runs from SRAM, initializes DRAM with the correct frequency and timing, then accepts a U-Boot image over UART and loads it into the now-working DRAM. From U-Boot you get a command line and can write corrected images to NAND.

## Investigation

MediaTek platforms have a UART rescue tool, `mtk_uartboot`, that pushes bl2 to the SoC's SRAM over serial for a temporary boot. I tried every 360T7 rescue bl2 and fip the community had shared. All failed with the same `dram size: 0MB`.

My first thought was that the 512MB chip had died. But someone on Xianyu was selling remote repair for 360T7 units with identical symptoms. If others fix it in software, it is not a hardware failure, and this router had run for years. I decided to work it out from source, kept the bl-mt798x repo that had always worked, and focused on DDR configuration.

I opened the router to confirm the chip: Zentel A3T4GF40ABF, DDR3-1600. The 360T7 default defconfig specifies no DDR3 frequency, and MT7981's Kconfig defaults to 2133:

```
# plat/mediatek/apsoc_common/Config.in
config _PLAT_MT7981
    select _DEFAULT_DDR3_2133
```

That confirmed the mismatch. OpenWrt's official bl2 and every community rescue package were compiled for DDR3-2133, so none could train this DDR3-1600 chip.

This was later fixed upstream. OpenWrt PR #22797, merged 2026-05-09, changes 360T7's BL2 from DDR3-2133 to DDR3-1866 and adds UBI layout support. I had flashed the latest OpenWrt stable at the time, which did not include the fix.

MT7981 ATF supports only two DDR3 frequency options: `_DDR3_FREQ_1866` (1866MT/s) and `_DDR3_FREQ_2133` (2133MT/s, default). There is no 1600 option. The lowest available is 1866, about a 17% overclock for this DDR3-1600 chip. Stepping a DDR3 chip up one speed bin is usually safe in practice.

With the frequency corrected to 1866, I compiled a bl2 and pushed it with mtk_uartboot. It timed out on the handshake. The default build takes the SPI-NAND boot path and omits the UART download handshake code. The source confirms bl2 needs both `_BOOT_DEVICE_RAM=y` and `_RAM_BOOT_RAM_BOOT_UART_DL=y` for mtk_uartboot. The first selects the RAM boot path and compiles `bl2_boot_ram.c`. The second enables the UART handshake and compiles `uart_dl.c`.

## Building the recovery images

Two images are needed. The first is a UART rescue bl2 for temporary boot from SRAM: RAM boot device, UART download handshake enabled, DDR3-1866. The second is a production bl2 and FIP for flashing to NAND: SPI-NAND boot, DDR3-1866.

### UART rescue bl2

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

Build it directly. The repo's `build.sh` hardcodes defconfig names and builds FIP, so it will not produce a UART-only bl2:

```bash
rm -rf atf-20240117-bacca82a8/build
make -C atf-20240117-bacca82a8 mt7981_360t7_uart_defconfig \
    CONFIG_CROSS_COMPILER=aarch64-linux-gnu- CROSS_COMPILER=aarch64-linux-gnu-
make -C atf-20240117-bacca82a8 bl2 \
    CONFIG_CROSS_COMPILER=aarch64-linux-gnu- CROSS_COMPILER=aarch64-linux-gnu- -j$(nproc)
```

Output is `build/mt7981/release/bl2.bin`. Rename it to `uart_bl2.bin`.

### Production bl2 and FIP

Modify `configs/mt7981_360t7_defconfig`:

```
_PLAT_MT7981=y
_MT7981_BOARD_BGA=y
_DRAM_DDR3=y
_DDR3_FREQ_1866=y
_LOG_LEVEL_INFO=y
```

Build with the repo script:

```bash
SOC=mt7981 BOARD=360t7 ./build.sh
```

Output is `output/mt7981_360t7-bl2.bin` (use as `flash_bl2.bin`) and `output/mt7981_360t7-fip-fixed-parts.bin` (use as `flash_fip.bin`).

## Recovery procedure

First push the rescue bl2 to SRAM over UART so it can initialize DRAM. Then push U-Boot into DRAM. From its prompt, use TFTP to load the corrected production images and write them to NAND.

Boot into U-Boot over UART:

```bash
mtk_uartboot -s /dev/ttyUSB0 -p uart_bl2.bin --aarch64 -f flash_fip.bin
```

Serial output shows `EMI: detected dram size: 512MB`, then drops to the `MT7981>` U-Boot prompt.

Flash over TFTP. Connect the PC to the router's LAN port. U-Boot comes up at `192.168.1.1`, so set the PC to `192.168.1.2/24` and run a TFTP server:

```
tftpboot 0x46000000 flash_bl2.bin
mtd erase bl2
mtd write bl2 0x46000000 0x0 $filesize

tftpboot 0x46000000 flash_fip.bin
mtd erase fip
mtd write fip 0x46000000 0x0 $filesize

reset
```

On reboot I flashed QWRT back, booted successfully, and the full 512MB was recognized.

## Files

I uploaded the compiled images. Because the DDR3 frequency is lowered to 1866MT/s, these should work for memory-modded 360T7 units whose chips cannot train at 2133:

[https://drive.qzydustin.com/360t7](https://drive.qzydustin.com/360t7)

Contents: `uart_bl2.bin`, `flash_bl2.bin`, `flash_fip.bin`. Follow the procedure above.

## Reference

### MTD partition layouts

Stock firmware, dual firmware partitions:

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

hanwckf bl-mt798x layout:

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

### Pitfall log

| # | Action | Result | Cause |
|---|--------|--------|-------|
| 1 | Flashed official OpenWrt bl2/fip per docs | Bricked, `dram size: 0MB` | Official firmware compiled for DDR3-2133, incompatible with 1600 chip |
| 2 | Community rescue bl2 | Same 0MB | Also 2133 frequency |
| 3 | Self-compiled DDR3-1866 bl2 + mtk_uartboot | Timeout | SPI-NAND boot mode, no UART_DL code |
| 4 | UART bl2 + DDR3-1866 + RAM boot | Success | Frequency and boot mode both correct |

### Links

- [mtk_uartboot](https://github.com/981213/mtk_uartboot): MediaTek SoC UART rescue tool
- [MediaTek Filogic Serial Rescue Guide](https://www.cnblogs.com/p123/p/18046679): MT7981/MT7986 walkthrough
- [bl-mt798x Source](https://github.com/hanwckf/bl-mt798x): ATF + U-Boot build
- [OpenWrt Wiki, Qihoo 360T7](https://openwrt.org/toh/qihoo/360t7_1.0): Official device page
- [OpenWrt PR #22797](https://github.com/openwrt/openwrt/pull/22797): DDR3 frequency fix + UBI layout

## NMBM compatibility

[digression: a second brick risk with the same root cause, mixing bootloader stacks]

During this rescue I hit another pitfall: NMBM, NAND Mapping Block Management. It is MediaTek's proprietary NAND bad-block layer. It reserves space at the end of flash for a mapping table and must be enabled or disabled consistently in both ATF (BL2) and the Linux kernel. A mismatch breaks partition offsets.

NAND bad blocks shift fixed offsets. Critical data sitting at raw fixed offsets, like the Factory calibration partition, can be lost if an earlier block goes bad and nothing remaps it. The firmware options handle this differently:

- NMBM (MediaTek proprietary, used by hanwckf): builds a bad-block mapping table for the whole NAND at the BL2 stage. Upper layers see a bad-block-free virtual device, and all partitions including Factory calibration are protected.
- OpenWrt default layout (old approach): fixed-partitions overall. Only the ubi partition holding rootfs and kernel has UBI bad-block management. Factory and fip sit at raw fixed offsets with no protection.
- OpenWrt UBI layout (new approach, PR #22797): no NMBM, but puts Factory and fip inside UBI volumes so all critical data is under bad-block management.

| Firmware/U-Boot | NMBM | Factory Protection | Partition Scheme |
|---|---|---|---|
| 360 stock | Enabled | NMBM transparent mapping | Dual firmware (ubi 36M + firmware-1 36M) |
| hanwckf bl-mt798x | Enabled | NMBM transparent mapping | Large ubi 108M + trailing partitions |
| OpenWrt default | Disabled | None (fixed-partitions, bad-block risk) | Large ubi 108M + trailing partitions |
| OpenWrt UBI (PR #22797, snapshot only) | Disabled | UBI volume management | Only bl2 1M fixed, rest in UBI volumes (127M) |

OpenWrt firmware is incompatible with hanwckf U-Boot. hanwckf's ATF enables NMBM, while OpenWrt's does not. NMBM reserves NAND space for its mapping table, so the two stacks see different partition sizes. Mixing them causes UBI volume-table checksum failures and can brick the router again. Flash bl2, fip, and firmware as a matching stack.

If your 360T7 runs hanwckf U-Boot, do not flash OpenWrt UBI-layout firmware alone. Either stay on immortalwrt-mt798x or QWRT with compatible firmware, or switch the entire stack to OpenWrt's official BL2 + FIP + firmware, and check the DDR frequency issue first.

## Firmware options for the 360T7

[digression: reference for choosing what to flash back, since the recovery above assumes you know]

| | OpenWrt Official | ImmortalWrt Official | hanwckf | QWRT (Lean) |
|---|---|---|---|---|
| Base | OpenWrt mainline | ImmortalWrt (OpenWrt fork) | ImmortalWrt 21.02 fork | Lean's LEDE |
| License | Open source | Open source | Open source (WiFi blob closed) | Closed source, paid |
| Distribution | Official site | Official site | GitHub source / community builds | QQ group, paid binary |
| Paired U-Boot | OpenWrt official U-Boot | ImmortalWrt official U-Boot | hanwckf bl-mt798x | hanwckf bl-mt798x |
| Kernel | 6.6 (24.10) | 6.6 (24.10) | 5.4 (locked to 21.02) | Undisclosed |
| WiFi Driver | Open source mt76 | Open source mt76 | MTK closed source | MTK closed source |
| Hardware Offload | Hardware flow offloading | Hardware flow offloading | MTK SDK native HNAT | MTK SDK native HNAT |

The open-source mt76 driver is still catching up to MTK's closed-source driver on signal strength, multi-client concurrency, and roaming. ImmortalWrt carries patches not accepted upstream, China-region packages, and a more aggressive kernel configuration. hanwckf is stuck on 21.02 because the MTK closed-source driver is tied to the old kernel SDK and cannot move to newer kernels. Mainline firmware uses hardware flow offloading for MT7981. hanwckf and QWRT use MTK SDK native HNAT, which is closer to the stock implementation.
