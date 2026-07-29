+++
title = "Setting Up an Nginx Web Server"
date = 2022-11-12T01:44:32-07:00

[taxonomies]
categories = ["DevOps"]
tags = ["nginx", "linux", "self-hosting"]
+++

Nginx is an event-driven web server. Its worker pool multiplexes I/O across many connections instead of assigning a thread to each one. It is commonly used for static sites, reverse proxies, and TLS termination.

<!--more-->

## Installation

Nginx ships in the package repositories of every mainstream Linux distribution. On Debian-based systems:

```shell
apt install nginx
```

## Process control

Nginx reads its configuration at startup and again on `reload`, so routine operation is signal-based rather than restart-based:

```shell
nginx -s stop # Fast shutdown
nginx -s quit # Graceful shutdown
nginx -s reload # Reload configuration file
nginx -s reopen # Reopen log files
```

`quit` lets in-flight requests finish, while `stop` drops them. Use `reload` after a configuration edit. It rereads the files without dropping existing connections.

Before any reload, validate the configuration. `nginx -t` parses the files and reports syntax errors without touching the running process. A failed reload leaves the old configuration active, but a typo you never tested leaves you debugging under pressure.

```shell
nginx -t
```

## How Nginx routes requests

Nginx first selects a `server` block by matching the request's `Host` header against `server_name`. It then selects a `location` block in that server by matching the request URI. Most configuration questions follow from those two choices.

The main file `/etc/nginx/nginx.conf` pulls in per-site files with:

```nginx
include /etc/nginx/sites-enabled/*.conf;
```

A site definition lives in `/etc/nginx/sites-available` and is enabled by symlinking it into `sites-enabled`:

```shell
ln -s /etc/nginx/sites-available/example.conf /etc/nginx/sites-enabled/
```

The symlink convention exists so that enabling and disabling a site is a filesystem operation rather than an edit to a shared file. Disable a site by removing the symlink, not by deleting the source.

## Serving static files

```nginx
server {
  listen 80;
  server_name domain.com;
  root /path/to/staticHtmlFile;
}
```

A `server` block defines one virtual host. `listen 80` binds the port. Nginx matches `server_name` against the incoming `Host` header to choose the block. `root` sets the directory Nginx appends the request URI to, so a request for `/index.html` resolves to `/path/to/staticHtmlFile/index.html`.

`root` and `alias` build paths differently. `root` appends the full request URI to its value. `alias` replaces the matched location prefix. A missing trailing slash on either the location or the alias can serve the wrong file or return a 404. Use `root` unless you need to strip a path prefix from the URI.

## TLS termination

```nginx
server {
  listen 443 ssl;
  server_name domain.com;
  ssl_certificate /location/certfile.crt;
  ssl_certificate_key /location/keyfile.key;
  root /path/to/staticHtmlFile;
}
```

Here Nginx terminates TLS. It holds the certificate, performs the handshake, and passes the decrypted request to an application server over HTTP on an internal port. Keeping TLS at the edge centralizes certificate renewal and HSTS.

## Reverse proxy

```nginx
server {
  listen 80;
  server_name domain.com;
  location / {
    proxy_pass https://target.com:port;
  }
}
```

A `location` block scopes directives to a URI prefix. `location /` matches every request, so the `proxy_pass` here forwards the entire site.

Nginx checks for an exact `=` match first. Otherwise it finds the longest matching prefix location. If that prefix carries `^~`, Nginx uses it and skips regex. Otherwise it tries regex locations (`~` case-sensitive, `~*` case-insensitive) in declaration order and uses the first match. If no regex matches, it falls back to the longest prefix. This ordering decides whether `/api` or `/static` handles a request when both blocks exist.

`proxy_pass` hands the request to an upstream and returns the response to the client. The upstream sees the request as coming from Nginx, not the browser, which is why a reverse proxy is also the natural place to set the forwarding headers an upstream app needs to identify the original client.
