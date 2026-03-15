# 📄 Paperless-ngx

Fully automated document management with OCR. Scan a document, it gets categorised, tagged, and made searchable in seconds. German bureaucracy finally under control.

- **URL:** `https://paperless.bhavibhavan.duckdns.org`
- **Host:** Portainer VM (`192.168.178.138:8010`)
- **Docs:** https://docs.paperless-ngx.com/

---

## 🏗️ Stack

- **paperless-ngx** — main webserver
- **PostgreSQL 17** — database (more reliable than default SQLite for production use)
- **Redis 8** — message broker for task queue

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
docker compose logs -f webserver
```

---

## ⚙️ Configuration

See [`stack.env.example`](./stack.env.example) for all variables. Key ones:

| Variable | Description |
|---|---|
| `PAPERLESS_SECRET_KEY` | Generate with `openssl rand -hex 32` — keep secret |
| `PAPERLESS_DBPASS` | PostgreSQL password — set same value in compose and env |
| `PAPERLESS_URL` | Your NPM proxy URL — required for correct redirects |
| `PAPERLESS_TIME_ZONE` | Set to your timezone |
| `PAPERLESS_OCR_LANGUAGE` | Primary OCR language (`eng`) |
| `PAPERLESS_OCR_LANGUAGES` | Additional languages (`deu` for German) |
| `PAPERLESS_FILENAME_FORMAT` | How files are stored: `correspondent/year/title` |
| `PAPERLESS_OCR_MODE` | `redo` — re-OCR all documents on import |

> ⚠️ **Important:** `PAPERLESS_DBPASS` must be set in both `stack.env` (as `PAPERLESS_DBPASS`) and is referenced in `docker-compose.yml` as `${PAPERLESS_DBPASS}` for the PostgreSQL container. Keep them in sync.

---

## 📥 Consuming Documents

Drop files into the `./consume` folder — Paperless picks them up automatically.

```bash
# Copy a document to consume
cp ~/Downloads/invoice.pdf ./consume/

# Watch logs to see it being processed
docker compose logs -f webserver
```

---

## 💾 Backup

```bash
# Export all documents (recommended before updates)
docker compose exec webserver document_exporter ../export

# The export folder is mounted at ./export
ls ./export/
```

---

## 🔄 Updates

```bash
docker compose pull
docker compose up -d
```