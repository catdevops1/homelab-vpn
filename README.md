# homelab-vpn

Self-hosted VPN using [Headscale](https://github.com/juanfont/headscale) on a VPS with a WireGuard exit node on a homelab bare-metal node. Provides a stable US IP for remote users routing through your home internet connection.

> This is the working solution after a failed attempt to run Headscale on Kubernetes behind Cloudflare Tunnel. See [k8s-headscale](https://github.com/catdevops1/k8s-headscale) for why that approach doesn't work.

---

## Architecture

```
Remote Client
        │
        │  Tailscale (WireGuard mesh)
        ▼
    VPS (Chicago)
    YOUR_VPS_IP
    Headscale + Caddy
        │
        │  WireGuard tunnel (outbound from homelab)
        ▼
    node01 (Homelab)
    Tailscale exit node
        │
        ▼
    US Internet
```

**What's exposed publicly:** VPS IP only. Home IP is never in DNS, never publicly visible.  
**What's encrypted:** All client↔VPS and VPS↔homelab traffic via WireGuard.  
**What's not routable:** Home LAN is not accessible from VPN clients — exit node only.

---

## Components

| Component | Where | Role |
|-----------|-------|------|
| Headscale v0.28.x | VPS (Ubuntu 24.04) | Control plane |
| Caddy | VPS | Reverse proxy + automatic TLS |
| UFW | VPS | Firewall (22, 80, 443 only) |
| Tailscale | node01 (homelab) | Exit node |
| Tailscale | Remote client | VPN client |

---

## VPS Setup

### 1. Install Headscale

```bash
# Download latest release
wget https://github.com/juanfont/headscale/releases/download/v0.28.0/headscale_0.28.0_linux_amd64.deb
apt install ./headscale_0.28.0_linux_amd64.deb -y
```

### 2. Configure Headscale

`/etc/headscale/config.yaml`:

```yaml
server_url: https://headscale.yourdomain.com
listen_addr: 127.0.0.1:8080
grpc_listen_addr: 127.0.0.1:50443

database:
  type: sqlite
  sqlite:
    path: /var/lib/headscale/db.sqlite

prefixes:
  v4: 100.64.0.0/10
  v6: fd7a:115c:a1e0::/48

dns:
  nameservers:
    global:
      - 1.1.1.1
      - 8.8.8.8
  magic_dns: true
  base_domain: vpn.internal

derp:
  server:
    enabled: false
  urls:
    - https://controlplane.tailscale.com/derpmap/default
  auto_update_enabled: true
  update_frequency: 24h

log:
  level: info
```

```bash
systemctl enable --now headscale
```

### 3. Install and Configure Caddy

```bash
apt install -y debian-keyring debian-archive-keyring apt-transport-https curl
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' > /etc/apt/sources.list.d/caddy-stable.list
apt update && apt install caddy -y
```

`/etc/caddy/Caddyfile`:

```
headscale.yourdomain.com {
    reverse_proxy 127.0.0.1:8080
}
```

```bash
systemctl enable --now caddy
```

### 4. Firewall (UFW)

```bash
ufw default deny incoming
ufw default allow outgoing
ufw allow 22/tcp
ufw allow 80/tcp
ufw allow 443/tcp
ufw enable
```

### 5. SSH Hardening

`/etc/ssh/sshd_config`:
```
PasswordAuthentication no
PermitRootLogin prohibit-password
PubkeyAuthentication yes
```

```bash
systemctl restart sshd
```

### 6. fail2ban

```bash
apt install fail2ban -y
systemctl enable --now fail2ban
```

---

## Headscale User & Key Setup

```bash
# Create a user
headscale users create myuser

# Generate a pre-auth key
headscale preauthkeys create --user myuser --reusable --expiration 24h
```

---

## Exit Node Setup (homelab node01)

```bash
# Install Tailscale
curl -fsSL https://tailscale.com/install.sh | sh

# Connect to your Headscale instance
tailscale up \
  --login-server https://headscale.yourdomain.com \
  --authkey <preauthkey> \
  --advertise-exit-node \
  --accept-routes

# Approve the exit node in Headscale
headscale routes list
headscale routes enable -r <route-id>
```

Enable IP forwarding on node01:
```bash
echo 'net.ipv4.ip_forward = 1' >> /etc/sysctl.conf
echo 'net.ipv6.conf.all.forwarding = 1' >> /etc/sysctl.conf
sysctl -p
```

---

## Client Setup (remote user)

```bash
# Install Tailscale on client machine
curl -fsSL https://tailscale.com/install.sh | sh

# Connect
tailscale up \
  --login-server https://headscale.yourdomain.com \
  --authkey <preauthkey> \
  --exit-node <node01-tailscale-ip> \
  --exit-node-allow-lan-access=false
```

---

## Security Model

| Surface | Status | Notes |
|---------|--------|-------|
| Home IP | ✅ Not exposed | Only VPS IP in DNS |
| Home LAN | ✅ Not routable | Exit node only, no subnet routes |
| VPS SSH | ✅ Key-only | Password auth disabled |
| VPS ports | ✅ Minimal | 22, 80, 443 only |
| VPN traffic | ✅ Encrypted | WireGuard end-to-end |
| TLS | ✅ Valid cert | Caddy + Let's Encrypt auto-renew |
| Brute force | ✅ Mitigated | fail2ban active |

**Note on exit node traffic:** All traffic from VPN clients exits through your home internet connection. You are responsible for what clients do with that access. Limit key sharing accordingly.

---

## Maintenance

```bash
# Check Headscale status
systemctl status headscale

# View connected nodes
headscale nodes list

# Rotate a pre-auth key
headscale preauthkeys create --user myuser --expiration 24h

# Update system
apt update && apt upgrade -y

# Check fail2ban
fail2ban-client status sshd
```

---

## DNS

Set an A record for `headscale.yourdomain.com` pointing to your VPS IP. **Do not proxy through Cloudflare** — leave it as DNS-only (gray cloud). Cloudflare's proxy is incompatible with Headscale's TS2021 protocol. See [k8s-headscale](https://github.com/catdevops1/k8s-headscale) for full details on why.
