# 🖥️ homelab

My personal self-hosted infrastructure — running on **Proxmox VE**, serving family & friends.

Everything here is documented so it can be reproduced from scratch. If you're starting your own homelab, feel free to use this as a reference.

---

## 🏗️ Infrastructure Overview

| Layer | Details |
|---|---|
| **Hypervisor** | Proxmox VE — 2 nodes |
| **Hardware** | Beelink mini PC |
| **DNS / Ad Blocking** | PiHole + Unbound (LXC container) |
| **Containers** | Docker via Portainer (dedicated VM) |
| **Reverse Proxy** | Nginx Proxy Manager |
| **Remote Access** | Tailscale VPN (zero-trust mesh) |
| **SSL** | DuckDNS + DNS-01 ACME challenge (no open ports) |

---

## 📦 Services

| Service | Description | Status |
|---|---|---|
| [💰 Actual Budget](./actual-budget/) | Privacy-first personal finance | ✅ Running |
| [📄 Paperless-ngx](./paperless-ngx/) | OCR document management | ✅ Running |
| [🛡️ PiHole + Unbound](./pihole-unbound/) | Network-wide ad blocking + recursive DNS | ✅ Running |
| [📸 Immich](./immich/) | Self-hosted Google Photos alternative | ✅ Running |
| [🎧 Audiobookshelf](./audiobookshelf/) | Audiobook & podcast server | ✅ Running |
| [🎬 Jellyfin](./jellyfin/) | Media streaming server | ✅ Running |
| [🔀 Nginx Proxy Manager](./nginx-proxy-manager/) | Reverse proxy + SSL termination | ✅ Running |
| [☸️ Kubernetes](./kubernetes/) | K8s cluster setup | 🚧 WIP |

---

## 🗂️ Repository Structure

```
homelab/
├── proxmox/               ← Proxmox setup: LXC, VMs, storage, network, PBS
├── pihole-unbound/        ← DNS ad blocking via community script + Unbound
├── actual-budget/         ← Docker Compose + env example
├── paperless-ngx/         ← Docker Compose + env example
├── immich/                ← Docker Compose + env example
├── audiobookshelf/        ← Docker Compose + env example
├── jellyfin/              ← Docker Compose + env example
├── nginx-proxy-manager/   ← Docker Compose + env example
└── kubernetes/            ← K8s manifests (WIP)
```

---

## 🔐 Security Approach

- All services are **private by default** — nothing exposed to the public internet
- Remote access via **Tailscale** mesh VPN only
- SSL certificates via **DuckDNS + DNS-01 ACME challenge** — no inbound ports required
- Secrets managed via **`.env` files** — never committed to git (see `.env.example` in each folder)

---

## 🚀 Getting Started

New to homelabs? Start here:

1. **[Proxmox setup](./proxmox/)** — install Proxmox, create VMs and LXC containers
2. **[PiHole + Unbound](./pihole-unbound/)** — set up DNS first, everything else depends on it
3. **[Nginx Proxy Manager](./nginx-proxy-manager/)** — reverse proxy before deploying apps
4. Then deploy any app from its folder

---

## ⚠️ Usage Notes

- All `.env.example` files show required variables — copy to `.env` and fill in your values
- Never commit your actual `.env` files
- Paths in compose files may need adjusting to match your setup

---

## 📫 About Me

**Ram Sambamurthy** — DevOps Engineer & Integration Architect· Stuttgart, DE

[![GitHub](https://img.shields.io/badge/github-satty008-0d1117?style=flat&logo=github&logoColor=white)](https://github.com/satty008)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-ramsambamurthy-0d1117?style=flat&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/ramsambamurthy/)