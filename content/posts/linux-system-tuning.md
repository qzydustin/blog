+++
title = "Linux System and Network Tuning"
date = 2022-07-28T15:02:35-07:00

[taxonomies]
categories = ["DevOps"]
tags = ["linux", "networking", "performance", "kernel"]
+++

Default Linux kernel parameters target general-purpose workloads. Servers handling high traffic, many connections, or heavy I/O need tuning. This guide covers the sysctl parameters that matter and why.

<!--more-->

## How sysctl works

Kernel parameters live in `/proc/sys`. `sysctl` reads and writes them. Persistent settings go in `/etc/sysctl.conf` or `/etc/sysctl.d/*.conf`.

```shell
sysctl net.ipv4.tcp_congestion_control          # read one value
sysctl -a | grep tcp                             # search all
sysctl -w net.ipv4.ip_forward=1                  # apply immediately (non-persistent)
sysctl --system                                  # load all config files
```

Changes take effect immediately. A reboot reloads from config files.

## TCP congestion control: BBR

CUBIC is the default congestion control. It treats packet loss as the congestion signal. On lossy links (Wi-Fi, cellular, cross-continent), CUBIC backs off too aggressively and never fills the pipe.

BBR (Bottleneck Bandwidth and RTT) models available bandwidth from RTT and delivery rate instead of loss. It sustains higher throughput on lossy paths.

```apacheconf
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr
```

BBR requires the `fq` (fair queue) pacing qdisc. Without it, pacing falls back to a less accurate method.

Check availability before enabling:

```shell
sysctl net.ipv4.tcp_available_congestion_control
# bbr may need: modprobe tcp_bbr
```

## Connection capacity

A listening socket has two queues. The SYN queue holds half-open connections (SYN received, ACK pending). The accept queue holds connections the kernel has finished but the application has not yet accepted.

```apacheconf
net.core.somaxconn = 32768                  # accept queue length
net.ipv4.tcp_max_syn_backlog = 16384       # SYN queue length
net.core.netdev_max_backlog = 32768        # packets queued before kernel processes them
```

When these queues fill, new connections drop or stall. nginx, haproxy, and similar servers must also raise their own listen backlog to match.

File descriptors cap total open sockets system-wide.

```apacheconf
fs.file-max = 1000000
```

Each process also has a limit. Check and raise it:

```shell
ulimit -n
# /etc/security/limits.conf:
# *  soft  nofile  1000000
# *  hard  nofile  1000000
```

Systemd services ignore limits.conf. Set `LimitNOFILE=1000000` in the unit file instead.

## TCP buffers

Larger buffers let the kernel hand more data to the network before waiting for ACKs. This matters on high-bandwidth, high-latency links (the bandwidth-delay product).

```apacheconf
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216
net.ipv4.tcp_moderate_rcvbuf = 1
```

The three values are min, default, max in bytes. `tcp_moderate_rcvbuf=1` lets the kernel auto-tune the receive buffer up to the max. Leave it on.

Buffer sizes are a tradeoff. Larger buffers use more memory per connection and can hide congestion. A million connections at 16 MB each is impossible. Match the setting to your connection count.

## Connection lifecycle

Outbound connections use ephemeral ports. The default range is small for a server making many outbound calls.

```apacheconf
net.ipv4.ip_local_port_range = 1024 65535
```

Closed sockets enter TIME_WAIT for up to 60 seconds. Under heavy load, TIME_WAIT accumulates and exhausts ports.

```apacheconf
net.ipv4.tcp_tw_reuse = 1          # recycle TIME_WAIT for new outbound connections
net.ipv4.tcp_fin_timeout = 15      # seconds in FIN-WAIT-2
net.ipv4.tcp_max_tw_buckets = 32768
```

`tcp_tw_reuse=1` is safe for outbound connections. Do not enable `tcp_tw_recycle` (removed in kernel 4.12). It broke NAT by misjudging connections sharing an IP.

Keepalive detects dead peers. Defaults wait two hours before probing. Lower them for services where dead connections must be reclaimed fast.

```apacheconf
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 3
```

TCP Fast Open skips a round trip on repeat connections. Value 1 enables client, 2 enables server, 3 enables both.

```apacheconf
net.ipv4.tcp_fastopen = 3
```

## Inotify limits

File watchers (inotify) back IDE indexing, tailing logs, and Node.js hot reload. The defaults are low. A large project can exceed them.

```apacheconf
fs.inotify.max_user_instances = 8192
fs.inotify.max_user_watches = 1048576
```

The symptom is `ENOSPC: System limit for number of file watchers reached`.

## Memory and swap

```apacheconf
vm.swappiness = 10
vm.dirty_ratio = 10
vm.dirty_background_ratio = 5
```

`swappiness` controls how eagerly the kernel swaps anonymous pages to disk. 0 avoids swapping anonymous pages except under memory pressure. 10 is a common server setting that keeps a small swap cushion. 60 (the default) suits desktops.

`dirty_ratio` and `dirty_background_ratio` set when dirty (unwritten) pages trigger writeback. High values batch writes but risk data loss on crash and cause latency spikes when the threshold hits. Lower values spread writes out.

```apacheconf
vm.overcommit_memory = 1
```

Value 1 lets the kernel overcommit memory (allocate more virtual than physical). Databases like Redis request this. Value 0 (default) uses heuristic overcommit. Value 2 is strict and can cause allocations to fail.

## Security hardening

A few parameters close common attack vectors.

```apacheconf
net.ipv4.tcp_syncookies = 1                  # survive SYN floods
net.ipv4.conf.all.rp_filter = 1              # drop spoofed source addresses
net.ipv4.conf.all.accept_redirects = 0       # ignore ICMP route redirects
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.all.accept_source_route = 0
```

`tcp_syncookies` keeps the SYN queue usable during a SYN flood by encoding connection state in the SYN-ACK instead of storing it. Enable it on any internet-facing host.

`rp_filter` (reverse path filtering) drops packets whose source address would not be routable back through the incoming interface, which blocks spoofed traffic. Multi-homed hosts may need `rp_filter=2` (loose mode).

## Applying changes

Write settings to a new file under `/etc/sysctl.d/` to keep them separate from distribution defaults.

```shell
sudo tee /etc/sysctl.d/99-tuning.conf <<EOF
# paste your settings here
EOF

sudo sysctl --system
```

Verify the values loaded:

```shell
sysctl net.ipv4.tcp_congestion_control
sysctl net.core.somaxconn
sysctl fs.file-max
```

These settings are starting points for a server under real load, not a universal profile. A host with 10 long-lived connections needs different buffers than one with 100,000 short-lived ones. Raising every value to the maximum wastes memory and can hurt latency. Measure before and after.
