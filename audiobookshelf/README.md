# 🎧 Audiobookshelf

Self-hosted audiobook and podcast server. Goodbye Audible subscription — full offline listening, progress sync across devices, and a polished mobile app.

- **URL:** `https://audiobookshelf.bhavibhavan.duckdns.org`
- **Host:** Portainer VM (`192.168.178.138:13378`)
- **Docs:** https://www.audiobookshelf.org/docs

---

## 🚀 Deployment

```bash
# Copy and fill in env file
cp stack.env.example stack.env
nano stack.env   # adjust paths if needed

# Create folders
mkdir -p audiobookshelf/{audiobooks,podcasts,config,metadata}

# Deploy
docker compose up -d

# Check status
docker compose ps
docker compose logs -f
```

---

## ⚙️ Configuration

See [`stack.env.example`](./stack.env.example):

| Variable | Description |
|---|---|
| `AUDIOBOOKS_LOCATION` | Path to audiobook files |
| `PODCASTS_LOCATION` | Path to podcast files |
| `CONFIG_LOCATION` | App config and database |
| `METADATA_LOCATION` | Covers and metadata cache |

---

## 📱 Mobile App

Install the Audiobookshelf app on Android/iOS and connect to `https://audiobookshelf.bhavibhavan.duckdns.org`. When outside the home network, connect Tailscale first.

---

## 🔄 Updates

```bash
docker compose pull
docker compose up -d
```

> ⚠️ Improvement added: `restart: unless-stopped` added to ensure the container restarts automatically after a Proxmox reboot or Docker daemon restart.