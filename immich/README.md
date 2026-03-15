# 📸 Immich

Self-hosted Google Photos alternative with ML-powered face recognition, automatic phone backup, and full search. All photos stay on-prem — no Google, no subscription.

- **URL:** `https://immich.bhavibhavan.duckdns.org`
- **Host:** Portainer VM (`192.168.178.138:2283`)
- **Docs:** https://immich.app/docs/

> ⚠️ Immich is under active development. Always check the [release notes](https://github.com/immich-app/immich/releases) before updating — breaking changes do happen.

---

## 🏗️ Stack

- **immich-server** — main application server
- **immich-machine-learning** — face recognition and CLIP search
- **PostgreSQL** (with pgvecto.rs) — database with vector search support
- **Valkey (Redis)** — job queue and caching

---

## 🚀 Deployment

```bash
# Copy and fill in env file
cp stack.env.example stack.env
nano stack.env

# Deploy
docker compose up -d

# Check status
docker compose ps
docker compose logs -f immich-server
```

---

## ⚙️ Configuration

See [`stack.env.example`](./stack.env.example) for all variables:

| Variable | Description |
|---|---|
| `UPLOAD_LOCATION` | Where photos are stored on the host (`./library`) |
| `DB_DATA_LOCATION` | Where PostgreSQL data is stored (`./postgres`) |
| `DB_PASSWORD` | PostgreSQL password — keep strong and secret |
| `IMMICH_VERSION` | Pin to a specific version or use `release` for latest |

---

## 📱 Mobile App

Install the Immich app on Android/iOS and point it to `https://immich.bhavibhavan.duckdns.org`. Enable auto-backup in the app settings.

When outside the home network, connect Tailscale first — then the app works seamlessly.

---

## 💾 Backup

Immich stores photos in `./library` — back this up regularly. The database is in `./postgres`.

```bash
# Backup database
docker compose exec database pg_dumpall -U immich > immich-db-backup.sql

# Restore database
docker compose exec -T database psql -U immich < immich-db-backup.sql
```

---

## 🔄 Updates

```bash
# Always check release notes first!
# https://github.com/immich-app/immich/releases

docker compose pull
docker compose up -d
```