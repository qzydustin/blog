+++
title = "Running BT and PT Behind CGNAT with WireGuard and a VPS"
date = 2024-11-11T19:52:11+00:00

[taxonomies]
categories = ["DevOps"]
tags = ["networking", "vps", "self-hosting", "wireguard"]
+++

BitTorrent and private-tracker clients benefit from accepting inbound connections, but CGNAT prevents that. Your ISP places multiple subscribers behind one public IPv4 address, so there is no inbound mapping for the home server. Port forwarding on the home router does not help because the relevant NAT is upstream at the ISP.

A VPS with a public address and a WireGuard tunnel can relay the traffic. It receives torrent connections on a port range and forwards them through the tunnel to the home server. To peers, the home server is then reachable at the VPS's public address.

<!--more-->

## Why reachability matters for BT and PT

A BitTorrent client announces a listen port to the tracker, and peers connect inbound to that port to pull pieces. A client with no reachable port can still dial out to peers that have one, but two NATed clients can never reach each other, so the pool of peers you can talk to shrinks to whichever happen to be publicly addressable.

Private trackers make this sting. Ratio enforcement assumes peers can reach you to upload. An announce port that will not accept inbound connections yields no upload credit, and a client that only connects outbound spends its time downloading.

## The relay design

The VPS holds the public address. The home server initiates the WireGuard UDP flow, which creates the tunnel through its NAT. WireGuard learns the home server's current endpoint from that flow and sends return traffic through it. The home NAT accepts those packets because they match an existing mapping. Forward the torrent port range with DNAT to the home server's tunnel address.

## VPS configuration

```ini
[Interface]
Address = 10.0.0.1/24, 2001:db8::1/64
ListenPort = 51820
PrivateKey = <VPS_Private_Key>

# Forward TCP/UDP traffic on ports 30000-39999 to the home server
PostUp = iptables -t nat -A PREROUTING -p tcp --dport 30000:39999 -j DNAT --to-destination 10.0.0.2
PostUp = iptables -t nat -A PREROUTING -p udp --dport 30000:39999 -j DNAT --to-destination 10.0.0.2
PostUp = iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostUp = ip6tables -t nat -A PREROUTING -p tcp --dport 30000:39999 -j DNAT --to-destination [2001:db8::2]
PostUp = ip6tables -t nat -A PREROUTING -p udp --dport 30000:39999 -j DNAT --to-destination [2001:db8::2]
PostUp = ip6tables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

PostDown = iptables -t nat -D PREROUTING -p tcp --dport 30000:39999 -j DNAT --to-destination 10.0.0.2
PostDown = iptables -t nat -D PREROUTING -p udp --dport 30000:39999 -j DNAT --to-destination 10.0.0.2
PostDown = iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
PostDown = ip6tables -t nat -D PREROUTING -p tcp --dport 30000:39999 -j DNAT --to-destination [2001:db8::2]
PostDown = ip6tables -t nat -D PREROUTING -p udp --dport 30000:39999 -j DNAT --to-destination [2001:db8::2]
PostDown = ip6tables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
PublicKey = <Home_Server_Public_Key>
AllowedIPs = 10.0.0.2/32, 2001:db8::2/128
```

The PostUp lines add DNAT rules for TCP and UDP on IPv4 and IPv6. MASQUERADE on eth0 rewrites the home server's outbound traffic to the VPS address. Without it, replies would be routed to the home server's unroutable tunnel address. PostDown removes those rules when the tunnel stops.

## Home server configuration

```ini
[Interface]
Address = 10.0.0.2/24, 2001:db8::2/64
PrivateKey = <Home_Server_Private_Key>

[Peer]
PublicKey = <VPS_Public_Key>
Endpoint = <VPS_IP>:51820
AllowedIPs = 0.0.0.0/0, ::/0
```

`AllowedIPs = 0.0.0.0/0, ::/0` routes all of the home server's traffic through the tunnel, including the torrent client's outbound connections. `Endpoint` is the VPS's public address on UDP 51820.

## Torrent client

Set the client's listening port range to 30000-39999 to match the DNAT. Do not configure a proxy in the client. With a proxy the reported port collapses to 0 and incoming connections fail.

## IPv6

Home ISPs that use CGNAT often do not provide IPv6 either, while a VPS usually has both protocols. The tunnel carries IPv6 alongside IPv4, so the home server can reach IPv6-only peers that were unavailable on the home connection.

## Caveats

If the tunnel goes idle, the carrier's NAT can drop the mapping, after which the VPS can no longer push inbound torrent connections back to the home server. `PersistentKeepalive = 25` in the home server's `[Peer]` block sends a heartbeat that holds the mapping open. Add it if your carrier reclaims idle flows aggressively.

WireGuard wraps each packet in UDP and its own header, so the tunnel MTU must sit below the underlying path MTU or large packets black-hole. WireGuard defaults to 1420, which works on most paths. If large transfers stall while small ones succeed, set `MTU = 1280` in both `[Interface]` blocks. 1280 is the IPv6 minimum and the safe fallback when path MTU discovery is broken.
