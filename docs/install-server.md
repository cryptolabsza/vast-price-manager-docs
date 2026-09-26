# Run VPM on your own server or VM

For running VPM somewhere other than your laptop — a small cloud VM, a
home server, a spare box. Same image, same compose file, different
network setup.

## What you need

- A small Linux server or VM: Ubuntu 24.04 or Debian 12.
- **amd64 (x86_64)** — the published image has no arm64 build, so an
  ARM-based VM (e.g. AWS Graviton, Oracle Ampere) won't pull it.
- Outbound internet access, to pull the image and reach Vast.ai's API.
- Resources, measured with `docker stats --no-stream` right after first
  login: about 85 MB RAM for VPM and 16 MB for Caddy (~100 MB total),
  well under 1% CPU at idle. About 350 MB of disk for the two container
  images, plus a small data volume (well under 1 MB at first run, grows
  slowly from there). Add about 180 MB more if you install the optional
  Vast CLI later — see [self-test.md](self-test.md).

## 1. Install Docker

Using Docker's own apt repository (not a piped install script):

```sh
# Add Docker's official GPG key:
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
```

```sh
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Enable Docker at boot:

```sh
sudo systemctl enable docker.service
sudo systemctl enable containerd.service
```

Let your user run `docker` without `sudo`, then log out and back in:

```sh
sudo usermod -aG docker "$USER"
```

Commands above are for Ubuntu, copied from Docker's own
[Ubuntu install page](https://docs.docker.com/engine/install/ubuntu/). On
Debian, follow Docker's [Debian install page](https://docs.docker.com/engine/install/debian/)
instead — same idea, one different repository URL.

## 2. Get the release files

```sh
sudo mkdir -p /opt/vpm && sudo chown "$USER" /opt/vpm && cd /opt/vpm
curl -fsSLO https://github.com/cryptolabsza/vast-price-manager-docs/releases/download/v0.3.1/compose.yml
curl -fsSLO https://github.com/cryptolabsza/vast-price-manager-docs/releases/download/v0.3.1/Caddyfile
curl -fsSLO https://github.com/cryptolabsza/vast-price-manager-docs/releases/download/v0.3.1/env.example
mv env.example .env
```

## 3. Choose how you'll reach VPM

### Option A — public domain, automatic HTTPS

For reaching VPM from anywhere.

1. Point a DNS A record for your domain at the VM's public IP.
2. Open 80 and 443 in the VM's own firewall:

   ```sh
   sudo ufw allow 80/tcp
   sudo ufw allow 443/tcp
   ```

   Also open both in your cloud provider's firewall or security group
   (AWS, GCP, Azure, DigitalOcean, …) — `ufw` alone won't help if the
   cloud network blocks the ports first.
3. In `Caddyfile`, replace the `localhost { tls internal ... }` block
   with the real-domain block, using your own domain instead of
   `vpm.example.com`.
4. In `.env`, set `VPM_ALLOWED_HOSTS` to that same domain.
5. Leave `HOST_HTTPS_PORT=443` and `HOST_HTTP_PORT=80` — Caddy needs both
   for the ACME certificate exchange.

### Option B — private only (LAN or VPN)

Nothing exposed to the internet. **Recommended for home labs** unless you
specifically need to reach VPM from outside your network.

1. Pick a hostname only your own network resolves, e.g. `vpm.lan`.
2. In `Caddyfile`, keep the `tls internal` block, but change `localhost`
   to your hostname.
3. In `.env`, set `VPM_ALLOWED_HOSTS` to that same hostname.
4. Your browser will warn about an untrusted certificate every time you
   connect — expected, since nothing outside your network can vouch for a
   private hostname's certificate. Click through it.

## 4. Set up and log in

From here it's the same as any other install:

1. Generate the master key and start the stack — [README.md quickstart, steps 3–4](../README.md#quickstart).
2. Log in and configure VPM — [first-run.md](first-run.md).

Nothing about running on a server changes those steps.

## Warnings

- **Never publish port 8088** (no `8088:8088` in `compose.yml`'s
  `ports:`). VPM expects to be reached only through Caddy's HTTPS port;
  publishing 8088 bypasses that entirely.
- **If this server is on the same local network as the machines you
  want to self-test, self-test will not work** — see
  [self-test.md](self-test.md) for why (NAT hairpinning) and what to do
  instead.
