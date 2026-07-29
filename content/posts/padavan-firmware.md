+++
title = "Building Padavan Router Firmware"
date = 2022-10-05T10:03:05-07:00

[taxonomies]
categories = ["Device"]
tags = ["firmware", "networking", "embedded"]
+++

[Padavan](https://bitbucket.org/padavan/rt-n56u/src/master/) is an open-source firmware derived from the GPL source ASUS released for the RT-N56U router. It targets MediaTek and Ralink SoCs: RT3883, MT7620, MT7621, and MT7628. The upstream repository on BitBucket last received updates in 2018. A fork on GitLab carries current development.

A prebuilt image targets one board and feature set. Building from source lets you match the device and select the features to compile. This post builds firmware for the MI-NANO router, whose official firmware is old and has no IPv6 support.

<!--more-->

## Build environment

Padavan's toolchain expects an older userspace. Debian Jessie is the recommended host because its package versions match what the 2018 source was tested against. Newer distributions can work, but building 2018-era source against a current toolchain tends to fail with errors that do not point back to the version mismatch.

Install the build dependencies and clone the source:

```shell
apt install build-essential gawk pkg-config gettext automake autogen texinfo autoconf libtool bison flex zlib1g-dev libgmp3-dev libmpfr-dev libmpc-dev git sudo vim module-init-tools
git clone https://bitbucket.org/padavan/rt-n56u.git /opt/rt-n56u
```

## Building the toolchain

Router SoCs are MIPS. Most build hosts are x86_64 or Arm. Padavan ships its own cross-toolchain recipe rather than relying on a distribution cross-compiler, so the version that builds the kernel matches the version that builds the userspace.

```shell
bash /opt/rt-n56u/toolchain-mipsel/build_toolchain
```

This step is slow but only runs once. Later firmware builds reuse the resulting toolchain.

## Configuration

The default config at `/opt/rt-n56u/trunk/.config` targets ASUS routers, with ASUS-specific templates in `/opt/rt-n56u/trunk/configs/templates`. For a non-ASUS board, copy the closest template, adjust it, and place board-specific files in `/opt/rt-n56u/trunk/board`.

The Padavan source has no MI-NANO configuration. The [padavan-ng](https://gitlab.com/padavan-ng/padavan-ng) fork on GitLab includes brand-specific configurations that help adapt the ASUS defaults.

The config you write selects the target SoC, wireless driver, package set, and kernel modules. IPv6, extra filesystem support, and specific drivers are toggled here rather than patched in after the fact.

## Building the firmware

Clean any stale artifacts, then build:

```shell
bash /opt/rt-n56u/trunk/clear_tree
bash /opt/rt-n56u/trunk/build_firmware
```

The finished image is written to `/opt/rt-n56u/trunk/images`. Flash it using the normal Padavan procedure for the device.
