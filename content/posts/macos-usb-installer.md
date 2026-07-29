+++
title = "Creating a Bootable macOS USB Installer"
date = 2023-07-19T17:06:26+00:00

[taxonomies]
categories = ["Device"]
tags = ["apple", "devices", "tooling", "macos"]
+++

Making a macOS USB installer becomes awkward when the host and target use different architectures or macOS generations. I ran into it while reinstalling macOS 10.7 Lion on a 2012 Mac Pro from an M1 MacBook Air. The M1 cannot run Lion's installer, and Apple's direct Lion download is packaged in a form that modern macOS refuses to open.

<!--more-->

## Downloading the installer

Apple provides direct downloads for 10.7 through 10.12 and App Store links for 10.13 and later. The older downloads arrive as a .dmg containing a .pkg. That package checks the host and refuses to run on an incompatible Mac. Instead, extract the Install app manually.

```shell
# Mount the dmg file
open -W ~/Downloads/InstallMacOSX.dmg
# Expand the pkg file
pkgutil --expand "/Volumes/Install Mac OS X/InstallMacOSX.pkg" ~/Lion
# Extract the app file
cd ~/Lion/InstallMacOSX.pkg
tar -xvf Payload
mv ~/Lion/InstallMacOSX.pkg/InstallESD.dmg ~/Lion/InstallMacOSX.pkg/Install\ Mac\ OS\ X\ Lion.app/Contents/SharedSupport
mv ~/Lion/InstallMacOSX.pkg/Install\ Mac\ OS\ X\ Lion.app ~/Downloads
# Cleanup
rm -rf ~/Lion
```

These commands never run the package installer. `pkgutil` expands the package, `tar` extracts the Install app from its Payload archive, and the two `mv` commands put InstallESD.dmg in the expected location before moving the finished app to ~/Downloads.

For 10.13 and later, the App Store is the official route, but it also refuses downloads for macOS versions the current Mac cannot run. [gibMacOS](https://github.com/corpnewt/gibMacOS) downloads installer assets from Apple's catalog without that check. Run gibMacOS.command to fetch them, then BuildmacOSInstallApp.command to assemble an Install app.

## Building the bootable image

[Eric from Canada's guide](https://ericfromcanada.github.io/output/2022/macos-installer-disk-images-for-virtualization.html) covers turning the Install app into a bootable disk image. That image is the source for the flashing step below.

## The standard path and why it fails here

Apple ships `createinstallmedia` inside each modern Install macOS app at Contents/Resources. On a supported Mac, format the USB and pass it to the tool. Because the tool runs on the host, the host must be able to execute it. Lion has no `createinstallmedia`; Apple introduced it with 10.9 Mavericks. A newer Intel binary may also fail on Apple Silicon or on a macOS release without its required runtime.

The USB must use a GUID partition scheme. Intel Mac firmware boots installers from GUID, not MBR or Apple Partition Map. The filesystem depends on the macOS version. Releases through 10.12 predate APFS and require HFS+ (Mac OS Extended, Journaled). Although 10.13 introduced APFS, Apple's documented `createinstallmedia` flow still formats the USB as Mac OS Extended (Journaled) with a GUID Partition Map, then lets the installer convert the target disk to APFS. An APFS USB will not boot a Mac installing 10.12 or earlier.

Apple's examples erase the USB to a fixed name and pass that name to `createinstallmedia` with `--volume`. The name itself is arbitrary. It only needs to match the erase command and the `--volume` path.

When `createinstallmedia` cannot run, flash a prebuilt image with `asr`.

## Flashing the image

1. Mount the bootable image.
2. Find the target USB with `diskutil list external`.
3. Restore the image to the USB with asr:

   ```shell
   sudo asr restore --source /dev/disk5 --target /dev/rdisk4 --erase
   ```

   Replace the device nodes with the ones `diskutil` reports for the mounted image and USB. The source is the image's regular device node. The target is the USB's raw device node, marked by the `r` prefix, for a block-level copy. Restoring the whole disk carries over the EFI and Recovery partitions a bootable USB needs. If the source is a single-partition image with no EFI partition, target the slice (`rdisk4s2`) instead. `--erase` wipes the target and reformats it to match the source image. Answer the prompts `asr` prints.

4. The USB is now bootable.

## Booting on modern systems

Whether a Mac will start from the stick depends on its firmware, and Apple tightened that twice. Pre-T2 Intel Macs, including the Mac Pro 2012, boot a GUID installer USB for a macOS version they support when you hold Option at startup and pick it. Macs with the Apple T2 Security Chip gate external boot behind firmware security. Enter Recovery, open Startup Security Utility, and set Secure Boot to No Security or Medium and allow booting from external media before the stick will start.

Apple Silicon Macs add two constraints. They boot only macOS versions built for their architecture, and Lion through Catalina are Intel-only, so they cannot start any installer this guide produces. For an installer an Apple Silicon Mac does support, hold the power button to reach startup options, select the volume, and authenticate with an admin account the first time.
