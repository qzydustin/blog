+++
title = "Remote LAN Access with ZeroTier, Tailscale, and Cloudflare Tunnel"
date = 2022-09-26T15:28:32-07:00

[taxonomies]
categories = ["DevOps"]
tags = ["networking", "vpn", "zerotier", "tailscale", "cloudflare", "self-hosting"]
+++

Remote LAN access lets devices reach a private network from anywhere. ZeroTier uses a virtual L2 network with P2P paths. Tailscale builds on WireGuard with NAT traversal. Cloudflare Tunnel uses a reverse proxy model.

<!--more-->

## ZeroTier: Virtual L2 overlay

ZeroTier creates a virtual Layer 2 network. Devices join a network ID and receive a 10.147.x.x address. Traffic prefers direct peer-to-peer paths. When NAT traversal fails, traffic relays through ZeroTier's infrastructure. A central controller manages authentication.

ZeroTier devices form a mesh. The controller only manages authentication. User traffic never touches it. NAT traversal uses hole-punching techniques similar to WebRTC. ZeroTier fits L2 requirements (broadcast, multicast) and self-hosted control planes. The virtual L2 model can confuse tools that expect physical network behavior.

### Installation and LAN exposure

1. Create network on [my.zerotier.com](https://my.zerotier.com). Note the 16-digit network ID.
2. Install on routing device:

```shell
curl -s https://install.zerotier.com | sudo bash
sudo zerotier-cli join <NETWORK_ID>
```

3. Authorize the device on my.zerotier.com.
4. Configure **Managed Routes** on my.zerotier.com:
   - Destination: your LAN subnet (e.g., `192.168.1.0/24`)
   - Via: the ZeroTier IP of the routing device (check with `sudo zerotier-cli listnetworks`)
5. On the routing device, enable IP forwarding and NAT:

```shell
sudo sysctl -w net.ipv4.ip_forward=1
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
sudo iptables -A FORWARD -i zt+ -o eth0 -j ACCEPT
sudo iptables -A FORWARD -i eth0 -o zt+ -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo apt install iptables-persistent
sudo iptables-save > /etc/iptables/rules.v4
```

6. Install ZeroTier on client devices. Join the same network ID.
7. Clients can now reach `192.168.1.0/24` through the routing device.

For home LANs, set the destination larger than the actual subnet (e.g., `192.168.0.0/16` for a `192.168.1.0/24` LAN). Physical interfaces take priority when on the same LAN.

## Tailscale: WireGuard with coordination

Tailscale builds on WireGuard, a Layer 3 VPN. Each device gets a 100.x.y.z address (CGNAT range). NAT traversal uses DERP relays when direct connection fails. A coordination server (proprietary, hosted by Tailscale) manages authentication and peer discovery.

Tailscale runs WireGuard between peers. The coordination server replaces manual key exchange. It maintains the list of peers, their public keys, and their endpoints. DERP relays work through restrictive firewalls using TCP. DERP traffic is encrypted end-to-end. The relay cannot inspect payloads, so it only sees encrypted blobs.

Peer Relay lets any node in your tailnet act as a relay for other peers. Configure high-speed nodes as relays and use ACLs to control which peers use them. This avoids "DERP roulette" where random DERP servers add latency.

### Installation and LAN exposure

1. Install on routing device:

```shell
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

2. Advertise subnet:

```shell
sudo tailscale up --advertise-routes=192.168.1.0/24
```

3. Approve the route on [Tailscale admin console](https://tailscale.com/admin/console).
4. Enable IP forwarding on the routing device:

```shell
sudo sysctl -w net.ipv4.ip_forward=1
sudo sysctl -w net.ipv6.conf.all.forwarding=1
```

5. Install Tailscale on client devices. Log in.
6. Clients can now reach `192.168.1.0/24` through the subnet router.

## Cloudflare Tunnel: Reverse proxy model

Cloudflare Tunnel (cloudflared) uses a reverse proxy architecture. The tunnel initiates outbound from your network to Cloudflare's edge. External traffic reaches your services through Cloudflare's network. No inbound ports or public IP required.

Cloudflare Tunnel runs a daemon (`cloudflared`) that maintains persistent outbound connections to Cloudflare's edge. Traffic flows from client to Cloudflare edge, then to your tunnel, then to your service. This differs from ZeroTier and Tailscale. Those tools create virtual networks where devices connect peer-to-peer. Cloudflare Tunnel places Cloudflare's network in the middle. Cloudflare Tunnel fits web services and Zero Trust policies with per-user ACLs. The reverse proxy model avoids inbound ports. Latency depends on Cloudflare edge proximity.

### Installation and LAN exposure (Web Dashboard)

1. Create tunnel on [Cloudflare Zero Trust Dashboard](https://dash.cloudflare.com/sign-up/zero-trust):
   - Zero Trust → Networks → Tunnels → Create a tunnel
   - Select "Cloudflared", enter name (e.g., `home-tunnel`)

2. Install cloudflared on routing device:

```shell
# Ubuntu/Debian
sudo apt update && sudo apt install cloudflared

# Or download binary
curl -L https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64 -o cloudflared
sudo mv cloudflared /usr/local/bin/cloudflared
sudo chmod +x /usr/local/bin/cloudflared
```

3. Copy the install command from the dashboard and run:

```shell
sudo cloudflared service install <TOKEN>
```

4. Verify:

```shell
systemctl status cloudflared
```

5. Add private route on dashboard:
   - Zero Trust → Networks → Routes → Add route
   - CIDR: `192.168.1.0/24`
   - Tunnel: `home-tunnel`

6. On client devices, install [Cloudflare WARP](https://1.1.1.1/). Log in to your Zero Trust organization.
7. Clients can now reach `192.168.1.0/24` through the tunnel.

## Comparison

| Aspect | ZeroTier | Tailscale | Cloudflare Tunnel |
|--------|----------|-----------|-------------------|
| Layer | L2 virtual LAN | L3 WireGuard | Reverse proxy (L7) |
| Model | Mesh P2P | Mesh P2P | Hub-and-spoke through Cloudflare |
| Client requirement | ZeroTier client | Tailscale client | WARP client |
| Control plane | Self-hostable | Proprietary (Headscale for self-host) | Cloudflare-hosted |
| NAT traversal | P2P hole-punching + relay | DERP relays + Peer Relay | Outbound-only (no inbound needed) |
| Best for | L2 needs, self-hosting | Zero-config mesh | Web services, Zero Trust |

Use ZeroTier for L2 requirements (broadcast, multicast) or self-hosted control planes. Use Tailscale for zero-configuration mesh networking with mobile support. Use Cloudflare Tunnel for web services with Zero Trust policies.
