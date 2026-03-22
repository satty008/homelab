# 🎬 Mediastack — Hodor (India NUC)

A full automated media stack running on **Hodor** — Intel NUC7i5BNH (i5, 16GB RAM, 256GB NVMe) in India. Accessible remotely via Tailscale through `debugndeadlift.duckdns.org`.

Jellyfin uses **Intel Quick Sync Video (QSV)** hardware transcoding — the NUC7i5BNH's integrated GPU handles transcoding without maxing out the CPU.

---

## 🏗️ Stack Overview

```
Requests (Jellyseerr)
        │
        ├── TV Shows → Sonarr
        ├── Movies   → Radarr
        ├── Books    → LazyLibrarian
        └── Comics   → Mylar3
                │
                ├── Prowlarr (indexer manager)
                │       └── FlareSolverr (Cloudflare bypass)
                │
                └── qBittorrent (download client)
                        │
                        ▼
                /data/downloads → /data/media
                        │
                        ▼
                Jellyfin (media server + QSV transcoding)
                Bazarr  (subtitles)
```

| Service | Port | Role |
|---|---|---|
| Jellyfin | 8096 | Media server |
| Audiobookshelf | 13378 | Audiobook, podcast & book server |
| qBittorrent | 49152 | Download client (WebUI) |
| Prowlarr | 9696 | Indexer manager |
| Sonarr | 8989 | TV show automation |
| Radarr | 7878 | Movie automation |
| Bazarr | 6767 | Subtitle automation |
| Whisparr | 6969 | Adult content automation |
| Mylar3 | 8090 | Comics automation |
| LazyLibrarian | 5299 | Books automation |
| Jellyseerr | 5055 | Media request UI |
| FlareSolverr | 8191 | Cloudflare bypass for indexers |
| Homepage | 8080 | nginx dashboard |

---

## 🗂️ Data Structure

All data lives under `/data/` on the Hodor VM:

```
/data/
├── config/                  ← app configs (one folder per service)
│   ├── jellyfin/
│   ├── audiobookshelf/
│   ├── sonarr/
│   ├── radarr/
│   └── ...
├── downloads/               ← qBittorrent download location
└── media/                   ← final media library
    ├── tv/
    ├── movies/
    ├── audiobooks/
    ├── podcasts/
    ├── books/
    ├── comics/
    └── adult/
```

> 💡 Using a single `/data/` path for both downloads and media means Sonarr/Radarr can **hardlink** files instead of copying them — saves disk space and is near-instant.

---

## 🌐 Network — npm_proxy

All services are on an external Docker network `npm_proxy` — shared with the NPM instance on Hodor (equivalent of Velocitail in DE).

```bash
# Create the network before deploying (only needed once)
docker network create npm_proxy
```

---

## ⚡ Hardware Transcoding — Intel QSV

Jellyfin uses Intel Quick Sync Video on the NUC7i5BNH for hardware-accelerated transcoding.

```yaml
devices:
  - /dev/dri:/dev/dri     # pass through Intel GPU
group_add:
  - "44"    # video group — check with: getent group video
  - "993"   # render group — check with: getent group render
```

### Verify QSV is working
```bash
# Check GPU is visible inside the container
docker exec jellyfin ls /dev/dri

# Check group IDs on your system (may differ from 44/993)
getent group video
getent group render
```

> ⚠️ Group IDs `44` (video) and `993` (render) are correct for this setup but may differ on other systems. Always verify with `getent group` before deploying.

### Enable QSV in Jellyfin UI
1. Admin → Dashboard → Playback → Transcoding
2. Hardware acceleration: **Intel QuickSync (QSV)**
3. Enable all supported codecs
4. Save and restart Jellyfin

---

## 🔗 Service Wiring

After deploying, connect the services in this order:

1. **Prowlarr** → add indexers, connect FlareSolverr for Cloudflare-protected ones
2. **Prowlarr** → sync to Sonarr, Radarr, Mylar3, LazyLibrarian, Whisparr
3. **qBittorrent** → configure download categories per service
4. **Sonarr/Radarr** → add qBittorrent as download client, set root folders
5. **Bazarr** → connect to Sonarr and Radarr
6. **Jellyseerr** → connect to Jellyfin + Sonarr + Radarr

---

## 🚀 Deployment

```bash
# Create network first
docker network create npm_proxy

# Create data directories
mkdir -p /data/{config,downloads,media/{tv,movies,audiobooks,podcasts,books,comics,adult}}
mkdir -p /data/config/{jellyfin,audiobookshelf,audiobookshelf/metadata,qbittorrent,sonarr,radarr,bazarr,prowlarr,whisparr,mylar3,lazylibrarian,jellyseerr,homepage}

# Deploy
docker compose up -d

# Check all services are running
docker compose ps
```

---

## 🔄 Updates

```bash
docker compose pull
docker compose up -d
```

---

## 📎 References

- [Jellyfin Docs](https://jellyfin.org/docs/)
- [Servarr Wiki](https://wiki.servarr.com/) — Sonarr, Radarr, Prowlarr, Bazarr
- [TRaSH Guides](https://trash-guides.info/) — quality profiles and best practices for the \*arr stack
- [Jellyseerr](https://github.com/Fallenbagel/jellyseerr)
- [LinuxServer.io](https://www.linuxserver.io/) — base images for most services