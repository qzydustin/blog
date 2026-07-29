+++
title = "Setting Up a Shadowsocks Proxy"
date = 2022-07-28T14:45:05-07:00

[taxonomies]
categories = ["DevOps"]
tags = ["proxy", "networking", "privacy", "censorship"]
+++

Shadowsocks-libev is an encrypted SOCKS5 proxy. It does not tunnel a whole network stack the way a VPN does. The client exposes a local SOCKS5 port that applications point at, and the server relays each connection to its destination over an encrypted link. The stream carries no recognizable protocol handshake, which is why it appears as random bytes to an observer. The usual topology puts the client behind a censoring network and the server on an overseas VPS.

This post covers the libev implementation on Debian-family systems. The Chinese censor does more than watch traffic passively. It actively probes suspected servers, and a confirmed Shadowsocks server has its IP blocked in mainland China. Cipher choice determines how long a server lasts, not whether it is invulnerable.

<!--more-->

## Cipher choice

Shadowsocks has two cipher families. Stream ciphers, the older family (`aes-256-cfb`, `rc4-md5`), provide confidentiality only. AEAD ciphers (`chacha20-ietf-poly1305`, `aes-256-gcm`) provide confidentiality and integrity by appending an authentication tag to each encrypted record.

Integrity is what helps against active probing. A censor can send its own probes to a suspected server. With a stream cipher, the server processes and responds to unauthenticated bytes, which can confirm that Shadowsocks is running. With an AEAD cipher, the tag fails on bytes the censor did not encrypt, so the server drops the connection. That is why this configuration uses `chacha20-ietf-poly1305`.

## Installing

Install the package:

```shell
apt install shadowsocks-libev
```

## Configuring

The configuration file lives at `/etc/shadowsocks-libev/config.json`. This example needs a unique password and port number:

```config
{
    "server":["::0", "0.0.0.0"],
    "mode":"tcp_and_udp",
    "server_port":8388,
    "local_port":1080,
    "password":"random-password",
    "timeout":86400,
    "method":"chacha20-ietf-poly1305"
}
```

`server` binds both IPv6 and IPv4. Setting `mode` to `tcp_and_udp` relays UDP alongside TCP, which lets DNS queries ride the tunnel. `server_port` is the port the censor sees. `local_port` is the SOCKS5 port the client exposes locally. `method` is the AEAD cipher from the previous section.

## Launching

Start the service:

```shell
systemctl enable shadowsocks-libev
systemctl restart shadowsocks-libev
systemctl status shadowsocks-libev
```

Connect to it with the [Outline](https://getoutline.org/) client.

## Entropy

On a fresh VPS the server may refuse to start, reporting:

```log
This system doesn't provide enough entropy...
```

The server needs a random-number source, and a fresh VPS may not have accumulated enough entropy. Install rng-tools and seed it from `/dev/urandom`:

```shell
apt install rng-tools
rngd -r /dev/urandom
systemctl restart shadowsocks-libev
```
