# 📝 Proxmox — Personal Notes & Gotchas

Things I've learned the hard way. Keeping them here so I don't repeat mistakes.

---

## 🔧 Post-Install Fixes

### Disable enterprise repo (no subscription)
Proxmox defaults to the enterprise repo which requires a paid subscription. Switch to the free repo:

```bash
# Disable enterprise repo
echo "# disabled" > /etc/apt/sources.list.d/pve-enterprise.list

# Add no-subscription repo
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" \
  > /etc/apt/sources.list.d/pve-no-subscription.list

apt update && apt upgrade -y
```

### Remove subscription nag popup
```bash
sed -Ezi.bak "s/(Ext.Msg.show\(\{[^}]+title: gettext\('No valid sub)/void\(\{\/\//g" \
  /usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js
systemctl restart pveproxy
```

---

## 🐧 LXC Gotchas

### LXC won't start after creation
- Check if the template downloaded fully — re-download if needed
- Make sure the assigned IP isn't already taken on the network
- Check logs: `pct start 100 && journalctl -u pve-container@100`

### Docker inside LXC (not recommended but possible)
If you need Docker inside an LXC (not recommended — use a VM instead):
```bash
# In the LXC config, add:
features: keyctl=1,nesting=1
# Edit via: nano /etc/pve/lxc/100.conf
```

### LXC loses network after Proxmox reboot
Usually a bridge issue. Check:
```bash
systemctl restart networking
brctl show
```

---

## 🖥️ VM Gotchas

### VM very slow — missing guest agent
Always install `qemu-guest-agent` inside VMs:
```bash
sudo apt install -y qemu-guest-agent
sudo systemctl enable --now qemu-guest-agent
```
Then in Proxmox UI: VM → Options → QEMU Guest Agent → Enable

### Can't see VM console in browser
- Try noVNC vs. xterm.js (VM → Console → dropdown)
- If blank screen: wait 30s after boot, guest agent needs to start

### VM clock drift
```bash
# Inside VM
sudo apt install -y chrony
sudo systemctl enable --now chrony
```

---

## 🌐 Network Gotchas

### Ubuntu 24.04 VM network config (Netplan)
Ubuntu VMs use Netplan instead of `/etc/network/interfaces`. Example from Velocitail:

```yaml
network:
  version: 2
  ethernets:
    ens18:
      match:
        macaddress: "bc:24:11:fa:a0:17"
      dhcp4: true
      dhcp6: true
      set-name: "ens18"
```

- `ens18` is the default interface name for VMs in Proxmox
- MAC address matching ensures the config applies to the right interface
- DHCP4+6 — static lease assigned on Fritz!Box by MAC address

Fritz!Box → Home Network → Network → IP Addresses → Add static IP lease

### vmbr0 not showing in new Proxmox install
After a fresh Proxmox install, `vmbr0` (the default Linux bridge) sometimes doesn't appear in the UI. This is a **Proxmox host level** issue — Proxmox runs Debian under the hood and uses the old `/etc/network/interfaces` file, not Netplan like Ubuntu VMs do.

```bash
# SSH into the Proxmox HOST (not a VM) and check
cat /etc/network/interfaces

# It should contain something like:
# auto vmbr0
# iface vmbr0 inet static
#     address 192.168.178.x/24
#     gateway 192.168.178.1
#     bridge-ports ens18
#     bridge-stp off
#     bridge-fd 0

# If vmbr0 is missing, add it manually then restart networking
systemctl restart networking
```

After restarting, refresh the Proxmox UI — `vmbr0` should appear under Node → Network.

---

## 💾 Storage Gotchas

### Disk full on local-lvm
```bash
# Check usage
pvesm status
df -h

# Clean up old CT templates and ISOs
# UI: Node → local → Content → delete unused items
```

### Can't delete a VM — disk stuck
```bash
# Force remove disk
lvremove /dev/pve/vm-100-disk-0
```

---

## 🔗 Tailscale on Proxmox

### Accessing Proxmox UI
Proxmox UI is accessed via the reverse proxy on Velocitail — no need to expose port 8006 directly:

```
https://pve-1.bhavibhavan.duckdns.org/
```

- **On home network (192.168.178.x)** → works directly, no Tailscale needed
- **Remote (outside home)** → enable Tailscale on your device first, then same URL works

This means a valid Let's Encrypt SSL cert via NPM — no self-signed cert warnings ever.

### Tailscale drops after Velocitail reboot
```bash
# SSH into Velocitail and check
systemctl status tailscaled
tailscale up --advertise-routes=192.168.178.0/24 --advertise-exit-node --accept-routes
```

---

## 📦 Misc Tips

- **Always take a snapshot before major changes** to a VM — right-click → Snapshots → Take Snapshot
- **Proxmox dark mode** — Profile (top right) → Color Theme → PVE Dark
- **Bulk actions** — you can start/stop/backup multiple VMs at once from Datacenter → select multiple
- **Resource mappings** — if passing through USB devices, use Datacenter → Resource Mappings for cleaner management