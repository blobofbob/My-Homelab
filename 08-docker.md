# 08 — Docker

Docker runs OpenClaw, n8n, and Vaultwarden as isolated containers. This guide installs Docker from Docker's official apt repository — the version in Debian's default repos (`docker.io`) is older and updated less frequently.

**Prerequisites:** [01 — Initial Setup](01-initial-setup.md) complete.

---

## Removing conflicting packages

Debian's default repositories include some Docker-related packages under different names. Remove them before adding Docker's own repo to avoid conflicts: [25]

```bash
for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do
  sudo apt-get purge $pkg
done
```

This is safe to run even if none of these are installed — apt will skip packages that aren't present.

---

## Adding the Docker repository

**1 — Install prerequisites:**

```bash
sudo apt-get update
sudo apt-get install ca-certificates curl -y
```

**2 — Add Docker's GPG key:**

```bash
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg \
  -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

**3 — Add the repository:**

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
  https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

---

## Installing Docker

```bash
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
```

Verify the installation:

```bash
sudo docker run hello-world
```

This pulls a test image, runs it, prints a confirmation message, and exits. If you see `Hello from Docker!`, Docker is installed and working correctly.

---

## Post-install — running Docker without sudo

By default, the `docker` command requires `sudo`. Add your user to the `docker` group to remove that requirement:

```bash
sudo usermod -aG docker $USER
```

Log out and back in for the group change to take effect. Verify:

```bash
docker run hello-world
```

This should work without `sudo` now.

> **Security note:** Users in the `docker` group can start containers that mount the host filesystem as root, which is effectively equivalent to `sudo` access. Only add trusted users to this group.

---

## Docker and the firewall

Docker manages its own iptables chains for container networking, independently of UFW. When a container stack starts, Docker inserts `DOCKER-USER` and `DOCKER-FORWARD` into the kernel's `FORWARD` chain, ahead of UFW's own rules.

> **Always restart running containers after enabling or reloading UFW.** On my system, a UFW enable/reload left Docker's per-container forwarding rules stale even though they still looked correct in `iptables -L DOCKER-FORWARD`. A full restart forces Docker to re-register its rules against the current firewall state and fixes it immediately.

`iptables-persistent` must never be installed alongside UFW — the two manage iptables independently, and installing it has been confirmed to remove UFW's own rules.

UFW is guide 11, set up last after all container stacks are already running.

---

## Maintenance

After pulling new images and restarting a stack, old image layers accumulate on disk. Remove them:

```bash
docker image prune -f
```

Run this after every `docker compose pull && docker compose up -d`. Without it, docker eats your disk space with old images.

---

## Useful commands

```bash
# Check Docker service status
sudo systemctl status docker

# List running containers
docker ps

# List all containers including stopped ones
docker ps -a

# List images
docker images

# Remove dangling images
docker image prune -f

# Remove all unused images (more aggressive)
docker image prune -a -f

# Check disk usage by Docker
docker system df
```

---

## Troubleshooting

**`docker: permission denied` after adding yourself to the docker group**

The group change requires a new login session to take effect. Log out and back in, or run `newgrp docker` in the current shell.

---

**`docker run hello-world` fails with a network error**

Docker may not have started. Check:

```bash
sudo systemctl status docker
sudo journalctl -u docker -n 30
```

---

**Next:** [11 — UFW](11-ufw.md)

**Sources:** [25]