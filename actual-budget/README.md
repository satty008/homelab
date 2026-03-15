# 💰 Actual Budget

Privacy-first personal finance — self-hosted, no subscriptions, no bank integrations phoning home. Envelope budgeting that actually works.

> My favourite self-hosted app. 🏆

- **URL:** `https://actual.bhavibhavan.duckdns.org`
- **Host:** Portainer VM (`192.168.178.138:5006`)
- **Docs:** https://actualbudget.org/docs/

---

## 🚀 Deployment

```bash
# On the Portainer VM, or via Portainer UI → Stacks → Add Stack
docker compose up -d

# Check status
docker compose ps
docker compose logs -f
```

No `.env` file needed — all config is in the compose file directly.

---

## ⚙️ Configuration

| Variable | Value | Description |
|---|---|---|
| `ACTUAL_UPLOAD_FILE_SYNC_SIZE_LIMIT_MB` | 200 | Max file sync size |
| `ACTUAL_UPLOAD_SYNC_ENCRYPTED_FILE_SYNC_SIZE_LIMIT_MB` | 200 | Max encrypted sync size |
| `ACTUAL_UPLOAD_FILE_SIZE_LIMIT_MB` | 200 | Max upload size |

---

## 💾 Data

Data is stored in a named Docker volume `actual_data` — managed by Docker, persists across container restarts and updates.

```bash
# Backup the volume
docker run --rm \
  -v actual_data:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/actual-backup.tar.gz /data

# Restore
docker run --rm \
  -v actual_data:/data \
  -v $(pwd):/backup \
  alpine tar xzf /backup/actual-backup.tar.gz -C /
```

---

## 🔄 Updates

```bash
docker compose pull
docker compose up -d
```