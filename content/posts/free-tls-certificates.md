+++
title = "Free Certificates with Certbot"
date = 2022-07-18T17:12:32-07:00

[taxonomies]
categories = ["DevOps"]
tags = ["security", "tooling", "self-hosting", "tls", "certificates", "lets-encrypt"]
+++

ACME is the protocol for automated certificate management. Let's Encrypt speaks ACME. Certbot is the official ACME client from EFF. This guide covers Certbot with Cloudflare DNS validation.

<!--more-->

## Setup

Install Certbot on Debian/Ubuntu:

```shell
sudo apt update
sudo apt install certbot python3-certbot-dns-cloudflare
```

On other systems, use snap:

```shell
sudo snap install certbot --classic
sudo snap install certbot-dns-cloudflare
```

Create a Cloudflare API token with Zone DNS edit permissions. Save it to a secure file:

```shell
echo "dns_cloudflare_api_token = YOUR_TOKEN" | sudo tee /etc/letsencrypt/secrets.ini
sudo chmod 600 /etc/letsencrypt/secrets.ini
```

Issue an ECC certificate:

```shell
sudo certbot certonly --dns-cloudflare \
  --dns-cloudflare-credentials /etc/letsencrypt/secrets.ini \
  --dns-cloudflare-propagation-seconds 30 \
  -d example.com -d *.example.com \
  --key-type ecdsa \
  --request-eff-email
```

Certificates save to `/etc/letsencrypt/live/example.com/`. Configure nginx:

```nginx
server {
    listen 443 ssl;
    server_name example.com;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
    ssl_prefer_server_ciphers off;
}
```

Reload nginx:

```shell
sudo nginx -t && sudo systemctl reload nginx
```

Certbot installs a systemd timer for auto-renewal. Check it:

```shell
sudo systemctl status certbot.timer
sudo certbot renew --dry-run
```

Add a deploy hook to reload nginx on renewal:

```shell
echo "renew_hook = systemctl reload nginx" | sudo tee -a /etc/letsencrypt/renewal-hooks/deploy/nginx.conf
```

## Certificate choice

Certbot supports RSA and ECDSA. Use ECDSA for better security per bit.

- `--key-type rsa`: RSA 2048-bit (default)
- `--key-type ecdsa`: ECDSA P-256 (recommended)
- `--key-type ecdsa --key-size 384`: ECDSA P-384 (higher security)

ECDSA support in older systems (Windows 7, Android < 7.0) is limited. Use RSA if you need broad compatibility.

## Root certificates and trust

Browsers and operating systems ship with trusted root CAs. Your certificate chains back to one of these roots.

Corporate environments deploy their own root CA. Employee devices trust this root. The corporate proxy then signs fake certificates for any HTTPS site. The employee's browser accepts these because the corporate root signed them.

```shell
# What a corporate proxy does:
# 1. Intercept connection to example.com
# 2. Generate fake certificate for example.com, signed by corporate root CA
# 3. Proxy connects to real example.com
# 4. Proxy decrypts, inspects, re-encrypts traffic
```

This breaks end-to-end encryption. Companies use it to inspect employee traffic. Parental control apps and antivirus software use the same mechanism.

Install a root CA certificate on Linux:

```shell
sudo cp corporate-root.crt /usr/local/share/ca-certificates/
sudo update-ca-certificates
```

On Windows, install via `certmgr.msc`. On macOS, use Keychain Access.

## Private key security

Private keys in `/etc/letsencrypt/live` are readable only by root. Certbot sets this automatically.

Do not expose keys in version control or logs. Check for leaked keys:

```shell
git log --all --full-history -- "*.key"
git log --all --full-history --source -- "*/privkey.pem"
```

Use tools like `trufflehog` or `git-secrets` before pushing.

If a key is compromised, revoke the certificate and issue a new one:

```shell
sudo certbot revoke --cert-path /etc/letsencrypt/live/example.com/cert.pem
sudo certbot delete --cert-name example.com
```

Let's Encrypt has rate limits. Too many reissues in short time will hit the limit.

## Rate limits

Let's Encrypt enforces rate limits to prevent abuse:

- 5 failed validations per hour
- 50 certificates per registered domain per week
- 5 duplicate certificates per week

Use the staging environment for testing:

```shell
sudo certbot certonly --dns-cloudflare --test-mode \
  --dns-cloudflare-credentials /etc/letsencrypt/secrets.ini \
  -d example.com
```

Staging certificates are not trusted by browsers. They do not count against rate limits.
