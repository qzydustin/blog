+++
title = "Running Proxmox on a 2012 Mac Mini"
date = 2024-11-11T19:29:42+00:00

[taxonomies]
categories = ["Device"]
tags = ["apple", "proxmox", "self-hosting"]
+++

The Late 2012 Mac Mini Server makes a cheap Proxmox VE node and is the last fully upgradeable Mac Mini. Its two SO-DIMM slots support 16GB of RAM, and its two 2.5-inch SATA bays can hold a boot disk and a datastore. Apple soldered the RAM in 2014 and moved the line to ARM in 2020, so newer Minis either cannot take more memory or cannot run x86 virtualization.

Its quad-core mobile Ivy Bridge i7 keeps power use modest for four cores. The whole machine is a 19cm-square box that fits on a shelf.

<!--more-->

## What you give up

A Mini is not a rack server. It has no BMC, so a hung kernel means walking over and holding the power button. It also has no ECC, and its power supply is not redundant. The smaller size and lower power use come at the cost of those safeguards.

## Booting on Apple hardware

Apple's firmware is a custom EFI, not the UEFI a PC uses, and it ignores the NVRAM entries `efibootmgr` writes. A stock Proxmox install boots from USB but may not come back on reboot. The fix is to bless the ESP's bootloader from macOS Recovery, or hold Option at the chime and pick the Linux EFI entry the first time. After that, GRUB on the ESP takes over normally.

## Drop the enterprise repo

Proxmox VE is Debian with a management layer that runs KVM virtual machines and LXC containers from a web UI. It ships with the enterprise APT repository enabled, which fails without a paid subscription. Disable it and switch to the no-subscription repo.

- Open the web UI at `https://ip:8006`
- Under **APT Repositories**, disable the enterprise repo and enable the no-subscription repo

## PVE Mods

[PVE Mods](https://github.com/Meliox/PVE-mods) extends the PVE web UI. On this Mini, I use it to remove the subscription nag and show temperature and fan RPM in the node summary. Follow the project's installation instructions.

## IOMMU and the Broadcom NIC

The onboard Broadcom NIC uses the `tg3` driver, which is flaky under Proxmox. Enabling Intel IOMMU can settle the NIC, and unlocks PCI passthrough for anything else you want to hand to a VM.

Edit the GRUB config:

```bash
sudo nano /etc/default/grub
```

Set the default command line:

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt"
```

Update GRUB:

```bash
sudo update-grub
```

Then reboot to apply the IOMMU settings.

If the NIC is still unstable, flip it to `intel_iommu=off`.

## Community scripts

The [Proxmox community scripts](https://community-scripts.github.io/ProxmoxVE/) collection installs helper tools in one line. I found these useful on the Mini:

- Set the [scaling governor](https://community-scripts.github.io/ProxmoxVE/scripts?id=scaling-governor) to `schedutil` so CPU frequency tracks load instead of sitting pinned.
- Use [LXC auto-update](https://community-scripts.github.io/ProxmoxVE/scripts?id=cron-update-lxcs) to keep containers patched without manual logins.
- Run [kernel clean](https://community-scripts.github.io/ProxmoxVE/scripts?id=kernel-clean) when old kernels fill the boot partition.
- Use [LXC log and cache clean](https://community-scripts.github.io/ProxmoxVE/scripts?id=clean-lxcs) to clear logs and cache accumulated by containers.
- Run [fstrim for LXC](https://community-scripts.github.io/ProxmoxVE/scripts?id=fstrim) to trim SSDs inside containers and preserve write performance.
- [Microcode](https://community-scripts.github.io/ProxmoxVE/scripts?id=microcode) is optional. The 2012 Mini sits on microcode `0x21`, which Intel and Apple have both stopped updating.

## Thermals

The Mini cools itself with one fan pulling air through a thermal chamber. At idle it is near-silent. Under sustained VM load it spins up audibly. Stock PVE shows no sensor data in the web UI, so without the PVE Mods panel you are flying blind on thermals.
