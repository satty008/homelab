# Moodle LMS on Proxmox with Docker Compose

[![Docker](https://img.shields.io/badge/Docker-Compose-blue?logo=docker&logoColor=white)](https://www.docker.com/)
[![Moodle](https://img.shields.io/badge/Moodle-LMS-orange?logo=moodle&logoColor=white)](https://moodle.org/)
[![MySQL](https://img.shields.io/badge/MySQL-8.4.5-blue?logo=mysql&logoColor=white)](https://www.mysql.com/)
[![Proxmox](https://img.shields.io/badge/Proxmox-VM-important?logo=proxmox&logoColor=white)](https://www.proxmox.com/)

This project deploys a **Moodle Learning Management System (LMS)** on a **Proxmox VM** using Docker Compose. It was designed to support an online exam setup for **200–500 concurrent users**.

The deployment includes a custom Moodle Docker image (built from the [Moodle Git repository](https://docs.moodle.org/500/en/Git_for_Administrators)), MySQL database, and phpMyAdmin for database management. **SSL termination and reverse proxying are handled by Nginx Proxy Manager** on Velocitail (separate Proxmox VM).

---

## 🚀 Project Overview

* **Moodle Service**: Custom Docker image (`satty008/runmoodle:main`) with an `entrypoint.sh` script that dynamically generates `config.php` using environment variables and secret files.
* **Database**: MySQL 8.4.5 with persistent storage via Docker named volumes.
* **phpMyAdmin**: Database management tool to interact with Moodle's MySQL database (port 8080).
* **External NPM & SSL**: Reverse proxy and SSL termination handled by Nginx Proxy Manager on Velocitail VM — not included in this compose stack.

---

## 🏗️ Architecture

The architecture of the deployment is documented in the [`architecture`](./architecture/) folder

<p align="center">
  <img src="./architecture/moodle-arch.png" alt="Moodle Architecture" width="600"/>
</p>

---

## 📦 Services

### 1. Moodle Service

* Custom Docker image (`satty008/runmoodle:main`) — built and pushed automatically via GitHub Actions.
* Runs Apache + PHP with Moodle codebase.
* Exposes port **8000** for access from Velocitail reverse proxy.
* Uses **entrypoint.sh** to auto-generate `config.php` at container startup.
* Persists uploaded files and cache in `moodleData` volume.

### 2. MySQL Database

* MySQL 8.4.5 container.
* Stores Moodle database.
* Persists data in `cloudDBdata` volume.
* Reads root and user passwords from secret files in `./secrets/`.
* Runs with `--mysql-native-password=ON` — required for MySQL 8.4 compatibility (see Troubleshooting).

### 3. phpMyAdmin

* Runs on port **8080**.
* Allows DB administrators to log in with MySQL credentials.
* Depends on `moodleDB` to be available.

### 4. External NPM & SSL (Not in this compose stack)

* Runs on **Velocitail VM** (`192.168.178.139`).
* Acts as reverse proxy and SSL terminator.
* Forwards traffic to Moodle service on `192.168.178.40:8000`.
* Domain: `moodle.bhavibhavan.duckdns.org` — private, accessible via Tailscale only.

---

## 🔑 Secrets & Configuration Management

### Environment Variables (`.env`)

The `.env` file contains non-sensitive database configuration:

```bash
MYSQL_DATABASE=moodledatabase
MYSQL_USER=moodleuser
MYSQL_HOST=moodleDB
MYSQL_PORT=3306
```

A template is provided as `.env.example`. Copy it to `.env`:

```bash
cp .env.example .env
```

### Secrets (Passwords)

Instead of storing passwords in `.env`, this setup uses **Docker secret files** in the `secrets/` folder:

```bash
mkdir secrets

# ⚠️ Always use single quotes — double quotes will strip $ characters from passwords!
echo 'your_secure_root_password' > secrets/db_root_password.txt
echo 'your_secure_db_password' > secrets/db_password.txt
```

**Important:** The `secrets/` folder is listed in `.gitignore` and must **never be committed** to version control.

---

## ⚙️ Deployment

### Prerequisites

* **Proxmox VM** running Ubuntu 24.04 LTS.
* Network connectivity to Velocitail VM for reverse proxying.
* Docker & Docker Compose installed.
* Domain name pointing to Velocitail NPM instance.

### Steps

1. **Clone Repository**

   ```bash
   git clone https://github.com/satty008/homelab.git
   cd homelab/moodle
   ```

2. **(Optional) Install Docker**

   If Docker is not installed:

   ```bash
   chmod +x proxmox-setup.sh
   ./proxmox-setup.sh
   ```

3. **Prepare Environment Variables**

   ```bash
   cp .env.example .env
   ```

   The `.env` file is pre-configured with non-sensitive values — no changes needed unless you want a different DB name or user.

4. **Create Secret Files**

   ```bash
   mkdir secrets

   # ⚠️ Use single quotes — double quotes strip $ from passwords!
   echo 'your_secure_root_password' > secrets/db_root_password.txt
   echo 'your_secure_db_password' > secrets/db_password.txt
   ```

5. **Start Services**

   ```bash
   docker compose up -d
   ```

6. **Run Moodle Database Installer**

   On first run, Moodle needs to install its database tables via CLI:

   ```bash
   docker exec moodle php /var/www/html/admin/cli/install_database.php \
     --agree-license \
     --adminpass='your_admin_password' \
     --adminemail="your@email.com" \
     --fullname="Your Site Name" \
     --shortname="siteshortname" \
     --lang=en
   ```

   > ⚠️ Use single quotes around `--adminpass` if your password contains `$`.

   On success you will see:
   ```
   ++ Success (x.xx seconds) ++
   Installation completed successfully.
   ```

7. **Configure NPM Reverse Proxy on Velocitail**

   In Nginx Proxy Manager UI (`https://velocitail.bhavibhavan.duckdns.org`):
   - **Domain:** `moodle.bhavibhavan.duckdns.org`
   - **Scheme:** `http`
   - **Forward IP:** `192.168.178.40`
   - **Port:** `8000`
   - **SSL:** select wildcard cert `*.bhavibhavan.duckdns.org` → Force SSL

8. **Access Services**

   * **Moodle LMS** → `https://moodle.bhavibhavan.duckdns.org`
   * **phpMyAdmin** → `http://192.168.178.40:8080`

---

## 🐛 Troubleshooting

### MySQL 8.4 — `Plugin 'mysql_native_password' is not loaded`

MySQL 8.4 removed `mysql_native_password` as a default authentication plugin. The `compose.yml` includes the fix via:

```yaml
command: --mysql-native-password=ON
```

If you see this error, ensure this line is present in the `moodleDB` service. Then wipe volumes and restart so MySQL reinitializes with the correct auth plugin:

```bash
docker compose down -v
docker compose up -d
```

> ⚠️ `-v` removes all volumes — only do this on a fresh install, not on a running instance with data.

### 500 Internal Server Error on fresh install

Moodle returns 500 until the database installer has been run. See step 6 above.

### Passwords with `$` are being stripped

Always use **single quotes** when creating secret files or passing passwords in bash:

```bash
# ✅ Correct
echo '$MyPassword' > secrets/db_password.txt

# ❌ Wrong — bash strips the $ and everything after it
echo "$MyPassword" > secrets/db_password.txt
```

### Change admin username

```bash
docker exec cloudDB mysql -u moodleuser -p'your_db_password' moodledatabase \
  -e "UPDATE mdl_user SET username='newusername' WHERE username='admin';"
```

---

## 🔄 Updates

Images are built and pushed automatically via GitHub Actions on every push to `main`. To update on the server:

```bash
docker compose pull
docker compose up -d
```

---

## 📝 Understanding `entrypoint.sh`

The `entrypoint.sh` script configures Moodle automatically when the container starts:

```bash
#!/bin/bash
set -e
```
* `set -e` — stop immediately if any command fails.

```bash
if [ ! -f "$MOODLE_CONFIG" ]; then
  DB_PASS="$(cat "$MOODLE_DATABASE_PASSWORD_FILE")"
```
* Checks if `config.php` already exists — generates it on first run only.
* Reads DB password securely from the mounted secret file.

```php
$CFG->sslproxy = true;
```
* Tells Moodle it's running behind an SSL-terminating proxy (NPM on Velocitail).

```bash
exec apache2-foreground
```
* Starts Apache in the foreground — keeps the container alive.

> In simple terms: **"On first run, generate Moodle's config from env vars and secrets, then start Apache. On later runs, skip config generation and just start Apache."**