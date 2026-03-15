# 🛡️ PiHole + Unbound

Network-wide ad blocking with **true recursive DNS** via Unbound, running in a lightweight **LXC container** on Proxmox.

---

## 🏗️ Architecture

```
Devices — get DNS 192.168.178.2 via Fritz!Box DHCP
        │
        │ DNS query
        ▼
Fritz!Box (192.168.178.1)
        │ DHCP announces PiHole as DNS server
        │ Local DNS: 192.168.178.2
        ▼
PiHole (192.168.178.2 · port 53)
        │ ad blocking + filtering
        │ passes allowed queries to Unbound
        ▼
Unbound (127.0.0.1 · port 5335)
        │ true recursive DNS — no upstream provider
        ▼
Root DNS servers → TLD servers → Authoritative servers
```

**Fritz!Box DNS config:**
- DHCP range: `192.168.178.20–200` — PiHole (`.2`) has a true static IP set directly on the LXC, outside the DHCP range. Velocitail (`.139`) is within the DHCP range but has a static lease assigned on the Fritz!Box by MAC address.
- All DHCP clients automatically get `192.168.178.2` as their DNS server — no per-device config needed
- Fritz!Box preferred DNS: `192.168.178.2` (PiHole) · fallback: `1.1.1.1` (Cloudflare) — internet still works if PiHole goes down

**Why this setup:**
- All devices get ad blocking automatically — including your wife's Mac without any Tailscale needed 😄
- Fritz!Box itself also queries through PiHole
- Unbound does **true recursive DNS** — no upstream provider sees your queries
- Private domain (`bhavibhavan.duckdns.org`) resolves locally without leaking upstream

---

## 📦 Installation

PiHole is installed via the [community scripts](https://community-scripts.org/scripts/pihole) project — a one-liner that creates and configures the LXC automatically.

### 1. Run the community script on the Proxmox host
```bash
# On the Proxmox host, run:
bash -c "$(wget -qLO - https://github.com/community-scripts/ProxmoxVE/raw/main/ct/pihole.sh)"
```

This will:
- Create a Debian LXC container
- Install PiHole inside it
- Configure basic settings

### 2. Install Unbound inside the LXC
The community script will ask if you want to install Unbound during setup — say **yes**. It will be installed and configured automatically alongside PiHole.

If you skipped it during setup, you can install it manually:

```bash
# Enter the PiHole LXC
pct enter <LXC ID>

# Install Unbound
apt install -y unbound

# Create the config file
nano /etc/unbound/unbound.conf.d/pi-hole.conf
# Paste the contents of unbound.conf in this folder

# Restart Unbound
systemctl restart unbound

# Test Unbound is working
dig google.com @127.0.0.1 -p 5335
```

### 3. Point PiHole to Unbound
In the PiHole admin UI (`http://192.168.178.2/admin`):
- Settings → DNS
- Uncheck all upstream DNS servers
- Set custom upstream DNS: `127.0.0.1#5335`
- Save

### 4. Point Fritz!Box to PiHole
In Fritz!Box UI:
- Home Network → Network → IPv4 Settings
- Scroll to **Local DNS server**
- Enter `192.168.178.2` (PiHole's IP)
- Save

Fritz!Box will now announce PiHole as the DNS server to **every device** that gets an IP via DHCP — no per-device configuration needed.

---

## ⚙️ Unbound Config

See [`unbound.conf`](./unbound.conf) for the full config. Key decisions:

| Setting | Value | Why |
|---|---|---|
| Interface | `127.0.0.1:5335` | Local only — PiHole talks to it, nothing else |
| Recursive DNS | no `forward-zone` block | Queries root servers directly — no upstream provider sees your queries |
| `hide-identity` | yes | Don't reveal server identity |
| `hide-version` | yes | Don't reveal Unbound version |
| `harden-dnssec-stripped` | yes | Reject DNSSEC-stripped responses |
| `qname-minimisation` | yes | Send minimal query info to each DNS server |
| `cache-min-ttl` | 300 | Cache responses for at least 5 min |
| `cache-max-ttl` | 14400 | Cache responses for at most 4 hours |
| `serve-expired` | yes | Serve stale cache while refreshing |
| `prefetch` | yes | Pre-fetch popular domains before TTL expires |
| `rrset-cache-size` | 256m | Large cache for better performance |
| Private addresses | RFC1918 blocked | Prevents DNS rebinding attacks |
| Private domains | `bhavibhavan.duckdns.org` (DE) · `debugndeadlift.duckdns.org` (India) | Resolved locally, not leaked upstream |

### Why true recursive DNS?
Unbound queries root servers directly — no third party (Google, Cloudflare, ISP) ever sees your DNS queries. Each query goes: root servers → TLD servers → authoritative servers. Maximum privacy, no upstream dependency.

To switch to DoT forwarding instead, add a `forward-zone` block to the config pointing to e.g. `8.8.8.8@853#dns.google`.

---

## ⚠️ Limitations — What PiHole Cannot Block

PiHole blocks ads at the **DNS level** — it works by blocking entire domains. This means it cannot block ads that are served from the same domain as the actual content.

| Service | Why PiHole can't block it |
|---|---|
| **YouTube** | Ads served from `youtube.com` and `googlevideo.com` — blocking these breaks YouTube entirely |
| **Spotify** | Ads served from same domains as music streaming |
| **Twitch** | Server-side ad injection — ads baked into the stream before it reaches you |
| **Facebook/Instagram** | Ads served directly from `facebook.com` |
| **Reddit** | Promoted posts served from `reddit.com` |
| **Hulu, Peacock** | Ads embedded server-side into the video stream |

**What PiHole IS great at:**
- Website banner and popup ads
- Tracking and telemetry scripts
- Smart TV and IoT device phone-home traffic
- Mobile app ads (many use separate ad domains)
- Windows telemetry
- General third-party ad networks

For YouTube specifically, browser extensions like **uBlock Origin** (desktop) or **ReVanced** (Android) are the practical solution.

```bash
# Check PiHole status
pihole status

# Update PiHole
pihole -up

# Update blocklists
pihole -g

# Check Unbound status
systemctl status unbound

# Test DNS resolution via Unbound directly
dig google.com @127.0.0.1 -p 5335

# Test DNS resolution via PiHole
dig google.com @192.168.178.2

# Check Unbound stats
unbound-control stats_noreset

# Restart both
systemctl restart unbound
pihole restartdns
```

---

## 🚧 SSH Access

The PiHole LXC is created via community script which does **not** set up SSH keys automatically — the `.ssh` directory exists but `authorized_keys` is empty by default.

### Add your SSH key

**Step 1** — Get your public key from your laptop:
```bash
cat ~/.ssh/id_ed25519.pub
```

**Step 2** — On the PiHole LXC via Proxmox console, add it:
```bash
cat >> /root/.ssh/authorized_keys << 'EOF'
ssh-ed25519 YOUR_PUBLIC_KEY_HERE your@email.com
EOF

chmod 700 /root/.ssh
chmod 600 /root/.ssh/authorized_keys
```

**Step 3** — Test from your laptop:
```bash
ssh -i ~/.ssh/id_ed25519 root@192.168.178.2
```

### Why not password auth?
The community script disables password login for root over SSH by default — key auth only. Check with:
```bash
grep -i "PasswordAuth\|PermitRoot" /etc/ssh/sshd_config
```

### Proxmox console as fallback
If SSH isn't working, always access via Proxmox UI:
```
https://pve-1.bhavibhavan.duckdns.org/ → PiHole LXC → Console
```
No SSH needed — direct root shell in the browser.