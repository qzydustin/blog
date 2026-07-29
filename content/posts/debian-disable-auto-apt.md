+++
title = "Disabling Automatic APT on a Low-Resource Debian VPS"
date = 2023-06-10T17:30:54+00:00

[taxonomies]
categories = ["DevOps"]
tags = ["linux", "vps", "self-hosting", "debian"]
+++

I run two inexpensive OranMe VPS instances: one has 0.1 CPU and 128MB RAM, the other 0.5 CPU and 512MB. They are only suitable for light self-hosted work, so I installed Debian without a desktop environment.

The 0.1 CPU box kept shutting down. In the OranMe control panel, its CPU usage hit the limit shortly before each crash. That looked like the provider's abuse protection killing the VM. `/var/log/system.log` showed that the spikes matched apt's periodic tasks.

<!--more-->

## What these timers do

Debian includes two systemd timers for apt. `apt-daily.timer` starts `apt-daily.service`, which runs `apt update` to refresh package lists. `apt-daily-upgrade.timer` starts `apt-daily-upgrade.service`, which passes security updates to `unattended-upgrades`. Both use a randomized delay so many machines do not hit package mirrors at once.

Automatic updates are useful on a machine with spare capacity. On a box with 0.1 CPU and 128MB RAM, they were the source of the trouble.

## Why it starves the box

A tenth of a core makes CPU-bound work take much longer. `apt update` parses and decompresses package indexes. `apt upgrade` runs `dpkg`, which decompresses archives and runs post-install scripts. Some of those scripts, especially man-db rebuilds and initramfs regeneration, need a fair amount of memory. With only 128MB, a brief spike can trigger the OOM killer. If it kills `dpkg`, the package database may be left half-configured.

There is also the lock. `dpkg` holds `/var/lib/dpkg/lock` for the whole upgrade. A manual apt command waits until the timer finishes. On a tenth of a core, that wait can make the server look stuck. The sustained CPU load during the upgrade is what triggered the provider's automatic shutdown.

## Disabling the timers

```shell
systemctl stop apt-daily-upgrade.service
systemctl disable apt-daily-upgrade.service
systemctl stop apt-daily-upgrade.timer
systemctl disable apt-daily-upgrade.timer

systemctl stop apt-daily.service
systemctl disable apt-daily.service
systemctl stop apt-daily.timer
systemctl disable apt-daily.timer
```

`stop` ends a running unit. `disable` removes the symlink that starts it at boot. Disabling the timers prevents future runs, while stopping the services clears any job already in progress. The order within a pair does not matter, but I stop the service before disabling its timer.

## Manual updates

Disabling the timers also disables automatic security patches. I would only do this on a lightly used server whose upgrade schedule I can manage myself. It is a poor choice for an internet-facing server.

You must run upgrades yourself, at a time when you can watch the server. First check what is pending:

```shell
apt list --upgradable
```

Then apply:

```shell
apt update
apt upgrade
```

Run the upgrade when you can watch CPU and memory use, rather than while the box is busy.
