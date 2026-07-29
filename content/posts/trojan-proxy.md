+++
title = "Setting Up a Trojan Proxy"
date = 2022-07-18T16:58:55-07:00

[taxonomies]
categories = ["DevOps"]
tags = ["trojan", "proxy", "networking", "privacy"]
+++

Trojan, from the trojan-gfw project, is a proxy protocol that uses TLS with a valid certificate on a real domain. Its traffic resembles ordinary HTTPS browsing.

The server fronts a web server with ordinary content. A probe receives that website instead of a dead end.

<!--more-->

## How Trojan handles probes

After the TLS handshake, the client sends the SHA-224 hash of the shared password as its first bytes. The server compares it with its configured passwords. If it matches, the rest of the connection becomes the proxied tunnel. If it does not, the server forwards the connection, including the hash bytes, to the fallback web server. The distinguishing bytes remain inside the encrypted TLS channel.

## Trojan versus Shadowsocks

Shadowsocks encrypts its payload but emits a byte stream without a protocol signature. Encryption defeats keyword matching, but censors have published statistical fingerprints based on entropy, packet sizes, and timing. Trojan sends its payload inside TLS records. It requires a real domain and a valid certificate, while Shadowsocks only requires a server.

## Setting it up

Install the package:

```shell
apt install trojan
```

The default configuration lives at /etc/trojan/config. It is JSON holding the shared password, the TLS certificate and key paths, the local listening port, and the fallback remote address that unauthenticated traffic forwards to. You need a random password and a valid certificate. Obtain the certificate for free with acme.sh against your domain, and generate the password as a random string shared between client and server.

Start and enable the service:

```shell
systemctl enable trojan
systemctl start trojan
systemctl status trojan
```

With the service running, point a client such as Shadowrocket at your domain and listening port using the same password.
