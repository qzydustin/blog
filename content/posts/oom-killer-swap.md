+++
title = "Surviving the OOM Killer with Swap"
date = 2024-01-13T03:27:26+00:00

[taxonomies]
categories = ["DevOps"]
tags = ["linux", "vps", "self-hosting", "memory"]
+++

The Linux OOM killer runs when the kernel can no longer reclaim memory. If anonymous memory exceeds RAM and has nowhere to go, the kernel scores processes and kills the one with the highest score. On a small VPS, that can be the application, database, or build job you meant to keep alive. A low-priority daemon with a large RSS can lose, but so can one large process when there is no swap.

Swap gives anonymous pages backing storage under memory pressure. The kernel can evict those pages instead of killing the owning process, at the cost of latency when it reads them back. On a VPS, that delay is often preferable to losing the process.

<!--more-->

## How the OOM killer picks a victim

The kernel scores each process by its memory footprint: RSS, page tables, and swap usage, with adjustments. Root-owned processes get a discount so system services are less likely to die. Niceness and runtime bias the score further. The result is exposed in `/proc/<pid>/oom_score` as a value from 0 to 1000. `oom_score_adj`, writable from -1000 to 1000, lets you override the heuristic. Set it to -1000 on a critical daemon and the OOM killer skips it entirely. Set it high on a disposable worker and it dies first.

That heuristic operates per memory domain. A global OOM scans the whole system. A cgroup OOM fires when a memory cgroup hits its limit. With `memory.oom.group` set, the killer is confined to processes inside that cgroup. If your app runs in a container or a systemd slice with a memory limit, it can be OOM-killed long before the host is under pressure.

## Why swap is a relief valve, not a performance feature

Without swap, anonymous memory is pinned in RAM. The kernel can reclaim file-backed pages by dropping clean caches or writing dirty ones back, but anonymous pages have no backing store, so they stay until OOM. Adding even a small swap file turns those pages into reclaim candidates. The kernel evicts the cold ones and keeps the hot ones resident.

Swap does not make a workload faster. Pages read back from disk are slow, and a workload that swaps constantly will stall. It helps when a workload only briefly exceeds RAM: the process survives and later recovers.

## Adding a swap file

Create a 1G swap file:

```bash
sudo fallocate -l 1G /swapfile
```

With `dd`, equivalent but slower:

```bash
sudo dd if=/dev/zero of=/swapfile bs=1024 count=1048576
```

Lock the file to root:

```bash
sudo chmod 600 /swapfile
```

Format and enable it:

```bash
sudo mkswap /swapfile
sudo swapon /swapfile
```

Persist it in `/etc/fstab` so it survives reboot:

```bash
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

## fallocate vs. dd

`fallocate` allocates space without writing zeros, so it is fast. `dd` writes zeros and is slower. Some filesystems, XFS included, recommend `dd` for swap files because of how `fallocate` reports space. On ext4 `fallocate` works. Check your filesystem before relying on it.

## Swappiness

`vm.swappiness` controls how readily the kernel reclaims anonymous rather than file-backed pages. It ranges from 0 to 100 and defaults to 60. A low value keeps anonymous pages resident longer, while a high value evicts them sooner. If swap is only a safety net, a lower value keeps it mostly idle until memory is tight. This is a preference, not a hard limit. The kernel can still swap at 0 under severe pressure.

## zram on a VPS

Disk swap on a VPS is slow. zram is a compressed block device in RAM: it trades CPU for effective capacity, and compression is faster than a disk round-trip. On a memory-constrained box a zram device absorbs cold pages without touching disk and without wearing the SSD.

zram does not add hard capacity. Its backing store is still RAM, so it extends the working set but cannot absorb a sustained overshoot the way disk swap can. A common setup pairs a small zram device for fast, compressed swap with a small disk swap file as last-resort overflow. zswap is the in-kernel equivalent: a compressed cache in front of a real swap device. Either is a better default than raw disk swap on a small VPS.
