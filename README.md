# 🖥️ homelab

My personal self-hosted infrastructure — running on **Proxmox VE** across 2 nodes (Beelink mini PC and an Intel NUC), serving family & friends.

Everything here is documented so it can be reproduced from scratch. If you're starting your own homelab, feel free to use this as a reference.

---

## 🏗️ Infrastructure Overview

| Layer | Details |
|---|---|
| **Hypervisor** | Proxmox VE — 2 nodes |
| **Hardware** | Beelink mini PC | Intel i5 NUC
| **DNS / Ad Blocking** | PiHole + Unbound (LXC container) |
| **Containers** | Docker via Portainer (dedicated VM) |
| **Reverse Proxy** | Nginx Proxy Manager |
| **Remote Access** | Tailscale VPN (zero-trust mesh) |
| **SSL** | DuckDNS + DNS-01 ACME challenge (no open ports) |

---

## 📦 Services

| Service | IP | Description | Status |
|---|---|---|---|
| [💰 Actual Budget](./actual-budget/) | 192.168.178.138:5006 | Privacy-first personal finance | ✅ Running |
| [📄 Paperless-ngx](./paperless-ngx/) | 192.168.178.138:8010 | OCR document management | ✅ Running |
| [🛡️ PiHole + Unbound](./pihole-unbound/) | 192.168.178.2 | Network-wide ad blocking + recursive DNS | ✅ Running |
| [📸 Immich](./immich/) | 192.168.178.138:2283 | Self-hosted Google Photos alternative | ✅ Running |
| [🎧 Audiobookshelf](./audiobookshelf/) | 192.168.178.138:13378 | Audiobook & podcast server | ✅ Running |
| [🎬 Jellyfin](./jellyfin/) | 192.168.178.22:8096 | Media streaming server | ✅ Running |
| [🔀 Velocitail](./velocitail/) | 192.168.178.139 | NPM reverse proxy + Tailscale gateway | ✅ Running |
| [🏠 Home Assistant](./home-assistant/) | 192.168.178.21:8123 | Home automation | 🚧 WIP |
| [☸️ Kubernetes](./kubernetes/) | — | K8s cluster setup | 🚧 WIP |

---

## 🗂️ Repository Structure

```
homelab/
├── proxmox/               ← Proxmox setup: LXC, VMs, storage, network, PBS
├── pihole-unbound/        ← DNS ad blocking via community script + Unbound
├── velocitail/            ← NPM reverse proxy + Tailscale subnet router
├── actual-budget/         ← Docker Compose + env example
├── paperless-ngx/         ← Docker Compose + env example
├── immich/                ← Docker Compose + env example
├── audiobookshelf/        ← Docker Compose + env example
├── jellyfin/              ← Docker Compose + env example
├── home-assistant/        ← Home Assistant VM (WIP)
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
3. **[Velocitail](./velocitail/)** — set up NPM + Tailscale before deploying apps
4. Then deploy any app from its folder

---

## ⚠️ Usage Notes

- All `.env.example` files show required variables — copy to `.env` and fill in your values
- Never commit your actual `.env` files
- Paths in compose files may need adjusting to match your setup

---

## 📫 About Me

**Ram Sambamurthy** — DevOps Engineer · Stuttgart, DE

[![GitHub](https://img.shields.io/badge/github-satty008-0d1117?style=flat&logo=github&logoColor=white)](https://github.com/satty008)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-ramsambamurthy-0d1117?style=flat&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/ramsambamurthy/)