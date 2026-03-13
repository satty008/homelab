# 🖥️ Proxmox Setup

My primary Proxmox node runs on a **Beelink Mini PC** in Stuttgart, DE. A second node — an Intel NUC in India — is connected via **Tailscale VPN** and currently a work in progress.

---

## 🏗️ Node Overview

| | Node 1 (Primary) | Node 2 (WIP) |
|---|---|---|
| **Hardware** | Beelink MINI-S12 (N100) | Intel NUC7i5BNH |
| **Location** | Stuttgart, DE | India |
| **CPU** | Intel Alder Lake-N100 (up to 3.4GHz) | Intel Core i5 |
| **RAM** | 16GB | 16GB |
| **Storage** | 256GB SSD · local-lvm + local (dir) | 256GB NVMe SSD |
| **Network** | vmbr0 (default bridge) | vmbr0 (default bridge) |
| **Remote Access** | Tailscale | Tailscale |
| **Status** | ✅ Running | 🚧 Work in Progress |

---

## 📦 Storage

Both nodes use the default Proxmox storage layout:

| Storage | Type | Used For |
|---|---|---|
| `local-lvm` | LVM-Thin | VM disks, LXC rootfs |
| `local` | Directory | ISO images, CT templates, backups |

### Downloading CT Templates
In the Proxmox UI:
1. Go to **Node → local → CT Templates**
2. Click **Templates**
3. Search for `debian` or `ubuntu` and download

Or via CLI:
```bash
pveam update
pveam available | grep -i debian
pveam download local debian-12-standard_12.7-1_amd64.tar.zst
```

---

## 🐧 Creating an LXC Container

LXCs are lightweight containers — ideal for simple services like PiHole, databases, or anything that doesn't need a full VM.

### Via Proxmox UI
1. Click **Create CT** (top right)
2. Fill in:
   - **Hostname** — e.g. `pihole`
   - **Password** — set a root password
   - **Template** — select your downloaded Debian/Ubuntu template
3. **Disk** — set size (4–8GB is usually enough for small services)
4. **CPU** — 1–2 cores
5. **Memory** — 256–512MB for lightweight services
6. **Network** — `vmbr0`, DHCP or static IP
7. **Confirm** and tick **Start after created**

### Via CLI
```bash
pct create 100 local:vztmpl/debian-12-standard_12.7-1_amd64.tar.zst \
  --hostname pihole \
  --memory 512 \
  --cores 1 \
  --rootfs local-lvm:8 \
  --net0 name=eth0,bridge=vmbr0,ip=dhcp \
  --start 1
```

### Useful LXC commands
```bash
pct list                    # list all containers
pct start 100               # start container 100
pct stop 100                # stop container 100
pct enter 100               # open shell inside container
pct config 100              # show container config
```

---

## 🖥️ Creating a VM

VMs are used for services that need a full OS — like the Docker/Portainer host.

### Via Proxmox UI
1. Click **Create VM** (top right)
2. **General** — set name, e.g. `docker-host`
3. **OS** — select ISO (upload first to `local → ISO Images`)
4. **System** — leave defaults (BIOS: SeaBIOS or OVMF for UEFI)
5. **Disks** — 20–32GB on `local-lvm`
6. **CPU** — 2–4 cores
7. **Memory** — 2048–4096MB depending on services
8. **Network** — `vmbr0`, VirtIO
9. **Confirm** → start and go through OS installer

### Uploading an ISO
```bash
# Via CLI
wget -P /var/lib/vz/template/iso/ https://releases.ubuntu.com/24.04/ubuntu-24.04-live-server-amd64.iso

# Or via UI: Node → local → ISO Images → Upload
```

### Useful VM commands
```bash
qm list                     # list all VMs
qm start 101                # start VM 101
qm stop 101                 # stop VM 101
qm reset 101                # hard reset VM 101
qm config 101               # show VM config
```

---

## 📋 Templates

Templates let you clone pre-configured VMs or LXCs instead of setting up from scratch every time.

### Create a VM Template
1. Set up a VM the way you want (install OS, update, install common tools)
2. Inside the VM, run:
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y qemu-guest-agent
sudo systemctl enable qemu-guest-agent
# Clean up
sudo cloud-init clean   # if cloud-init is installed
sudo truncate -s 0 /etc/machine-id
sudo poweroff
```
3. In Proxmox UI: right-click the VM → **Convert to Template**
4. To deploy: right-click template → **Clone** → Full Clone

### Create an LXC Template
```bash
# After configuring an LXC, convert it to a template
pct template 100
# Clone from template
pct clone 100 101 --hostname new-container --full
```

---

## 🌐 Network — vmbr0

Both nodes use the default `vmbr0` Linux bridge. All VMs and LXCs connect to it and get IPs from the home router (FRITZ!Box) via DHCP, with static DHCP leases assigned per MAC address for consistency.

```
Internet
   │
Fritz!Box (192.168.178.1)
   │
vmbr0 (Proxmox bridge)
   ├── VM: docker-host    (192.168.178.x)
   ├── LXC: pihole        (192.168.178.2)
   └── LXC: ...
```

### View network config
```bash
# Ubuntu 24.04 VMs use Netplan (not /etc/network/interfaces)
sudo cat /etc/netplan/*.yaml

# Check IP addresses and interfaces
ip addr show
```

---

## 🔗 Remote Access via Tailscale — Velocitail

Rather than installing Tailscale on every VM, LXC, or the Proxmox host itself, I run a dedicated VM called **Velocitail** that acts as a **Tailscale subnet router and exit node**.

**Velocitail** (`192.168.178.139` · Tailscale: `100.87.172.94`) runs:
- **Tailscale** — advertises the entire local subnet (`192.168.178.0/24`) to the Tailscale mesh, making every VM and LXC automatically reachable without installing Tailscale on each one. Also acts as an **exit node** so remote devices can route traffic through the home network.
- **Nginx Proxy Manager** — reverse proxy for all services via `bhavibhavan.duckdns.org`

```
Tailscale Mesh (100.x.x.x)
        │
   Velocitail VM · 192.168.178.139 · 100.87.172.94
   ├── Tailscale subnet router → advertises 192.168.178.0/24
   ├── Exit node → remote devices route traffic via home network
   └── Nginx Proxy Manager → bhavibhavan.duckdns.org/* → services
        │
   vmbr0 · Fritz!Box · 192.168.178.0/24
   ├── docker-host VM     → Immich, Paperless, Actual Budget, etc.
   ├── pihole LXC         → 192.168.178.2
   └── other VMs/LXCs     → all reachable via Tailscale automatically
```

### Tailscale network
| Device | Tailscale IP | Role |
|---|---|---|
| velocitail | 100.87.172.94 | Subnet router + exit node (Stuttgart, DE) |
| hodor | 100.117.150.11 | Subnet router + exit node (India NUC — WIP) |

**Why this approach:**
- Install Tailscale **once** — entire subnet reachable, no per-VM setup
- Proxmox host itself doesn't need Tailscale installed
- Exit node lets remote devices browse as if on the home network
- Clean separation — all networking concerns live in one VM

### Setting up a Tailscale subnet router + exit node
```bash
# Inside Velocitail VM (Ubuntu 24.04)
curl -fsSL https://tailscale.com/install.sh | sh

# Enable IP forwarding (required)
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# Advertise subnet and enable exit node
tailscale up --advertise-routes=192.168.178.0/24 --advertise-exit-node --accept-routes
```

Then in the **Tailscale admin console**:
- Approve the advertised subnet route for Velocitail
- Approve the exit node

---

## 💾 Backups — Planned (PBS)

Backups are not yet configured. **Planned:** Set up **Proxmox Backup Server (PBS)** on a dedicated VM or external machine.

PBS will provide:
- Incremental backups (only changed blocks)
- Deduplication across backups
- Scheduled backup jobs per VM/LXC
- Restore from Proxmox UI directly

```bash
# Planned backup schedule
# - Daily incremental backups of all VMs and LXCs
# - Retention: 7 daily, 4 weekly
# - Target: dedicated PBS instance
```

> 📌 This section will be updated once PBS is set up.

---

## 🔧 Useful Proxmox CLI Commands

```bash
# Node info
pvesh get /nodes
pveversion

# Storage info
pvesm status

# List all VMs and LXCs
qm list && pct list

# Check resource usage
pvesh get /nodes/<nodename>/status

# Manage services
systemctl status pve-cluster
systemctl status pvedaemon
```