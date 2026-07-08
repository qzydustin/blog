+++
title = "Docker and Containers: Security, Rootless Mode, and Podman"
date = 2022-07-07T20:36:36-07:00

[taxonomies]
categories = ["DevOps"]
tags = ["linux", "docker", "containers", "podman", "self-hosting", "rootless", "security"]
+++

A container is not a virtual machine. VMs emulate hardware and run a full OS kernel. Containers share the host kernel and isolate processes using namespaces (which partition what a process sees: filesystem, network, PIDs, users) and cgroups (which cap what it uses: CPU, memory, I/O).

<!--more-->

Docker provides an image format, a registry (Docker Hub), a CLI, and a daemon that manages container lifecycle. Docker Compose extends this to multi-container applications defined in a YAML file.

## Installation

The fastest path on Debian/Ubuntu systems:

```shell
curl -fsSL https://get.docker.com | sh
```

This adds the official Docker apt repository and installs the current stable release. For production systems, use the explicit apt method:

```shell
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
  https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

Docker Compose is now a CLI plugin (`docker compose`). The standalone `docker-compose` binary is deprecated.

## Root access and bind mounts

The Docker daemon runs as root. Bind-mounting host files into a container gives the container process write access to them on the host.

```shell
docker run --rm -it -v /etc/passwd:/etc/passwd alpine sh
```

Inside the container, `/etc/passwd` is the host's file. Adding a user with UID 0:

```shell
echo 'hacked::0:0::/root:/bin/sh' >> /etc/passwd
```

On the host, `su hacked` opens a root shell. The Docker socket (`/var/run/docker.sock`) is worse. Many users mount it into containers for CI/CD tools or Docker-in-Docker setups. Any process with socket access controls the daemon, which runs as root.

## Rootless mode

Rootless mode runs the daemon and containers under your user account. The daemon starts via systemd user services and stores its socket at `$XDG_RUNTIME_DIR/docker.sock`. Container processes map to user-level UIDs via `/etc/subuid` and `/etc/subgid`. Root inside the container (UID 0) maps to an unprivileged UID on the host (typically 100000+), so even with container root the process cannot modify host files.

```shell
sudo apt-get install uidmap
dockerd-rootless-setuptool.sh install
export DOCKER_HOST=unix://$XDG_RUNTIME_DIR/docker.sock
```

Add the export to your shell profile. The `/etc/passwd` bind-mount attack no longer works because the container process is unprivileged on the host.

Network performance is lower (slirp4netns handles NAT instead of the kernel bridge), and `--privileged` containers are more restricted. For self-hosting workloads these rarely matter.

## Podman

Podman is a daemonless container runtime. Each container runs as a direct child process of the calling user with no central daemon. Rootless operation is the default.

```shell
sudo apt-get install podman

podman pull nginx
podman run -d -p 8080:80 nginx
podman ps
```

Compose workflows work via `podman-compose`:

```shell
sudo apt-get install podman-compose
podman-compose up -d
```

Docker rootless still runs a user-owned daemon. Compromising that daemon compromises all containers. Podman has no shared supervisor. Each container is an independent process. Some Docker-specific features (BuildKit, Docker Desktop integration, certain volume drivers) do not work with Podman. For standalone containers and compose stacks on Linux servers, Podman is functionally equivalent.

Docker gives best ecosystem support and is required for Docker Desktop or BuildKit-dependent builds. Docker rootless offers the same ecosystem with better security. Podman offers no daemon and no shared supervisor to compromise.

## Further isolation

Rootless mode and Podman reduce the daemon's privilege but containers still share the host kernel. For multi-tenant environments where untrusted code runs (serverless platforms, CI services), this is insufficient. Firecracker (used by AWS Lambda) wraps each container in a lightweight VM with its own kernel. gVisor (used by GKE Sandbox) places a userspace kernel between the container and the host. It filters syscalls before they reach the host kernel. Both trade performance for stronger isolation. For single-tenant self-hosting, the complexity cost outweighs the benefit.
