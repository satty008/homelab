# Moodle on Proxmox VM - Setup Guide

## Step 1: Prepare the Proxmox VM

SSH into your Proxmox VM (Ubuntu 24 LTS):

```bash
ssh user@<proxmox-vm-ip>
```

## Step 2: Install Docker (if not already installed)

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER
newgrp docker
```

Verify Docker is working:
```bash
docker --version
docker compose version
```

## Step 3: Clone the Repository

```bash
cd /opt  # or your preferred location
git clone https://github.com/satty008/homelab.git
cd homelab/moodle
```

## Step 4: Set Up Configuration Files

### Create secrets directory:
```bash
mkdir secrets
echo "your_secure_root_password" > secrets/db_root_password.txt
echo "your_secure_db_password" > secrets/db_password.txt
chmod 600 secrets/*.txt  # Restrict permissions
```

### Verify .env is set:
```bash
cat .env
# Should show:
# MYSQL_DATABASE=moodledatabase
# MYSQL_USER=moodleuser
# MYSQL_HOST=moodleDB
# MYSQL_PORT=3306
```

## Step 5: Build the Docker Image

```bash
docker build -t satty008/runmoodle:1.7 .
```

Verify the build:
```bash
docker images | grep satty008/runmoodle
```

## Step 6: Start the Services

```bash
docker compose up -d
```

Check if containers are running:
```bash
docker compose ps
```

Expected output:
```
moodle     Up
moodleDB   Up
phpmyadmin Up
```

## Step 7: Verify Services

### Check Moodle:
```bash
curl http://localhost:8000
# Should return HTML (Moodle page)
```

### Access phpMyAdmin:
Open browser: `http://<proxmox-vm-ip>:8080`
- Username: `moodleuser` (from .env)
- Password: from `secrets/db_password.txt`

## Step 8: Configure External Nginx Reverse Proxy

On your **external Nginx VM**, add this config:

```nginx
# /etc/nginx/sites-available/moodle.conf

upstream moodle_backend {
    server <moodle-vm-ip>:8000;
}

server {
    listen 80;
    server_name moodle.bhavibhavan.duckdns.org;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name moodle.bhavibhavan.duckdns.org;

    ssl_certificate /path/to/your/cert.pem;
    ssl_certificate_key /path/to/your/key.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    location / {
        proxy_pass http://moodle_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
        proxy_connect_timeout 600s;
        proxy_send_timeout 600s;
        proxy_read_timeout 600s;
    }
}
```

Enable and reload Nginx:
```bash
sudo ln -s /etc/nginx/sites-available/moodle.conf /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

## Step 9: First-Time Moodle Setup

Visit: `https://moodle.bhavibhavan.duckdns.org`

Moodle will detect the database and run the installer:
- Set admin username/password
- Configure site settings
- Complete the setup wizard

## Troubleshooting

### Check container logs:
```bash
docker compose logs -f moodle
docker compose logs -f moodleDB
```

### Stop all services:
```bash
docker compose down
```

### Remove all data (WARNING - deletes database):
```bash
docker compose down -v
```

### Verify database connection:
```bash
docker compose exec moodleDB mysql -u moodleuser -p -h localhost moodledatabase
# Enter password from secrets/db_password.txt
```

## Persistence

Your data is stored in Docker volumes:
- `moodleData`: Moodle files & uploads
- `cloudDBdata`: MySQL database

These survive container restarts and are backed up if you backup the volume paths.

## Monitoring

Check container status:
```bash
docker stats
docker compose ps
```

View logs in real-time:
```bash
docker compose logs -f
```

## Backup Strategy

Backup the volumes regularly:
```bash
docker run --rm -v moodle_moodleData:/data -v /backup:/backup_dest alpine tar czf /backup_dest/moodleData-$(date +%Y%m%d).tar.gz /data

docker run --rm -v moodle_cloudDBdata:/data -v /backup:/backup_dest alpine tar czf /backup_dest/cloudDBdata-$(date +%Y%m%d).tar.gz /data
```

---

**Next Steps:**
1. Prepare your Proxmox VM with Docker
2. Clone the repo
3. Create secrets
4. Build and start services
5. Configure external Nginx
6. Access `https://moodle.bhavibhavan.duckdns.org`
