# 🔀 Velocitail — Reverse Proxy + Tailscale Gateway

Velocitail is a dedicated Ubuntu 24.04 VM on Proxmox that acts as the **network gateway** for the entire homelab. It runs two things:

1. **Nginx Proxy Manager** — reverse proxy with automatic SSL via Let's Encrypt
2. **Tailscale** — subnet router and exit node for the entire `192.168.178.0/24` network

All homelab services are accessed via `bhavibhavan.duckdns.org` subdomains, privately over Tailscale — no ports exposed to the public internet.

---

## 🏗️ Architecture

```
Remote device (with Tailscale)
        │
        │ https://service.bhavibhavan.duckdns.org
        ▼
Tailscale mesh (100.x.x.x)
        │
        ▼
Velocitail · 192.168.178.139 · 100.87.172.94
        │
        ├── Tailscale subnet router → 192.168.178.0/24
        │
        └── Nginx Proxy Manager (port 443)
                │
                ├── pve.bhavibhavan.duckdns.org           → 192.168.178.10:8006    (Proxmox)
                ├── velocitail.bhavibhavan.duckdns.org    → 192.168.178.139:81     (NPM Admin)
                ├── portainer.bhavibhavan.duckdns.org     → 192.168.178.138:9443   (Portainer)
                ├── immich.bhavibhavan.duckdns.org        → 192.168.178.138:2283   (Immich)
                ├── paperless.bhavibhavan.duckdns.org     → 192.168.178.138:8010   (Paperless-ngx)
                ├── actual.bhavibhavan.duckdns.org        → 192.168.178.138:5006   (Actual Budget)
                ├── audiobookshelf.bhavibhavan.duckdns.org→ 192.168.178.138:13378  (Audiobookshelf)
                ├── jellyfin.bhavibhavan.duckdns.org      → 192.168.178.22:8096    (Jellyfin)
                ├── homeassistant.bhavibhavan.duckdns.org → 192.168.178.21:8123    (Home Assistant)
                ├── pihole.bhavibhavan.duckdns.org        → 192.168.178.2:80       (PiHole)
                └── fritzbox.bhavibhavan.duckdns.org      → 192.168.178.1:80       (Fritz!Box)
```

**Key points:**
- `bhavibhavan.duckdns.org` is a **private domain** — only reachable on the Tailscale subnet
- Valid **Let's Encrypt SSL certificates** via DNS-01 challenge — no open ports required
- On the home network (`192.168.178.x`) — works without Tailscale
- Remote access — connect Tailscale first, then all URLs work

---

## 📦 Nginx Proxy Manager

### docker-compose.yml
See [`docker-compose.yml`](./docker-compose.yml) — uses SQLite (default, no separate DB needed for a homelab).

### Start NPM
```bash
cd ~/nginxproxy
docker compose up -d

# Check it's running
docker compose ps
docker compose logs -f
```

### Admin UI
```
http://192.168.178.139:81
```

Default credentials on first run:
```
Email:    admin@example.com
Password: changeme
```
**Change these immediately after first login.**

### Adding a Proxy Host
1. Login to NPM admin UI
2. **Hosts → Proxy Hosts → Add Proxy Host**
3. Fill in:
   - **Domain name** — e.g. `immich.bhavibhavan.duckdns.org`
   - **Scheme** — `http` (NPM handles SSL termination)
   - **Forward Hostname/IP** — IP of the service e.g. `192.168.178.X`
   - **Forward Port** — port of the service e.g. `2283`
   - Tick **Block Common Exploits**
4. **SSL tab** → Request a new SSL certificate → tick **Force SSL**
5. Save

---

## 🔐 SSL — Let's Encrypt via DNS-01 Challenge

### Why DNS-01 instead of HTTP-01?
The standard Let's Encrypt HTTP-01 challenge requires port 80 to be open to the internet. Since `bhavibhavan.duckdns.org` is a **private domain**, HTTP-01 would fail.

DNS-01 proves domain ownership by creating a TXT record in DNS — no inbound ports needed.

### How it works
```
NPM requests cert for *.bhavibhavan.duckdns.org
        │
        ▼
Let's Encrypt asks NPM to create a DNS TXT record
        │
        ▼
NPM uses DuckDNS API token to create the TXT record
        │
        ▼
Let's Encrypt verifies the TXT record
        │
        ▼
Certificate issued ✅ — valid for 90 days, auto-renewed
```

### Setting up DNS-01 in NPM
1. In NPM → **SSL Certificates → Add SSL Certificate → Let's Encrypt**
2. Domain: `*.bhavibhavan.duckdns.org` (wildcard covers all subdomains)
3. Tick **Use a DNS Challenge**
4. DNS Provider: **DuckDNS**
5. Enter your **DuckDNS API token** (from `https://www.duckdns.org` → your account)
6. Request Certificate

Once issued, select this certificate when adding proxy hosts — all subdomains share one wildcard cert.

---

## 🔗 Tailscale — Subnet Router + Exit Node

Tailscale is installed directly on Velocitail, not via Docker. It advertises the entire home subnet so every VM and LXC is reachable via Tailscale without installing Tailscale on each one.

### Installation
```bash
curl -fsSL https://tailscale.com/install.sh | sh

# Enable IP forwarding (required for subnet routing)
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# Start Tailscale as subnet router + exit node
tailscale up \
  --advertise-routes=192.168.178.0/24 \
  --advertise-exit-node \
  --accept-routes
```

### Approve in Tailscale admin console
1. Go to `https://login.tailscale.com/admin/machines`
2. Find Velocitail → **Edit route settings**
3. Approve `192.168.178.0/24` subnet route
4. Approve exit node

### Useful commands
```bash
# Check Tailscale status
tailscale status

# Check advertised routes
tailscale status --json | jq '.Self.AllowedIPs'

# Restart Tailscale
sudo systemctl restart tailscaled

# Re-advertise after reboot (if not persistent)
tailscale up --advertise-routes=192.168.178.0/24 --advertise-exit-node --accept-routes
```

### Make advertising persistent across reboots
```bash
# Check if tailscaled is enabled
sudo systemctl is-enabled tailscaled

# Enable if not already
sudo systemctl enable tailscaled
```

---

## 🔧 Useful Commands

```bash
# NPM logs
cd ~/nginxproxy && docker compose logs -f

# Restart NPM
cd ~/nginxproxy && docker compose restart

# Update NPM
cd ~/nginxproxy
docker compose pull
docker compose up -d

# Check open ports
ss -tlnp | grep -E '80|81|443'

# Check Tailscale
tailscale status
tailscale ping 192.168.178.2   # ping PiHole via subnet route
```

---

## 📎 References

- [Nginx Proxy Manager — Official Docs](https://nginxproxymanager.com/guide/)
- [Nginx Proxy Manager — Full Setup](https://nginxproxymanager.com/setup/)
- [DuckDNS](https://www.duckdns.org)
- [Tailscale Subnet Routers](https://tailscale.com/kb/1019/subnets)
- [Tailscale Exit Nodes](https://tailscale.com/kb/1103/exit-nodes)