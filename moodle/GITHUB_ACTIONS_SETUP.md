# GitHub Actions CI/CD Setup for Moodle Docker Image

This workflow automatically builds and pushes the Moodle Docker image to DockerHub whenever you push to the main/develop branches or create a release.

## Setup Instructions

### Step 1: Create DockerHub Access Token

1. Go to [Docker Hub](https://hub.docker.com)
2. Sign in to your account
3. Click your profile icon → **Account Settings** → **Security** → **Access Tokens**
4. Click **New Access Token**
5. Name it: `github-actions` (or similar)
6. Permissions: `Read & Write`
7. Copy the token (you won't be able to see it again)

### Step 2: Add Secrets to GitHub

1. Go to your repo: `https://github.com/satty008/homelab`
2. Settings → **Secrets and variables** → **Actions**
3. Click **New repository secret**

Add these two secrets:

| Secret Name | Value |
|-------------|-------|
| `DOCKERHUB_USERNAME` | Your DockerHub username (e.g., `satty008`) |
| `DOCKERHUB_TOKEN` | The access token you just created |

### Step 3: Verify Setup

Push a change to trigger the workflow:

```bash
git add .github/workflows/docker-build.yml
git commit -m "ci: add GitHub Actions for Docker image build and push"
git push
```

Check the workflow:
1. Go to repo → **Actions** tab
2. Click **Build and Push Moodle Docker Image**
3. Watch the build progress

## How It Works

The workflow triggers on:
- ✅ Push to `main` or `develop` branches
- ✅ Changes to Dockerfile, entrypoint.sh, or compose.yml
- ✅ Release creation (tags like v1.7)

### Tagging Strategy

Images are tagged as:
- `satty008/runmoodle:main` - Latest from main branch
- `satty008/runmoodle:develop` - Latest from develop branch
- `satty008/runmoodle:1.7` - Release version
- `satty008/runmoodle:main-abc123def` - Commit SHA
- `satty008/runmoodle:latest` - Always latest from main

## Deploying to Proxmox VM

Once the image is in DockerHub, your Proxmox VM deployment becomes super simple:

### Option 1: Manual Pull & Run

```bash
# SSH into Proxmox VM
ssh user@<proxmox-vm-ip>

# Go to your moodle directory
cd homelab/moodle

# Pull the latest image
docker pull satty008/runmoodle:main

# Stop old containers
docker compose down

# Start with new image (it will use the one you just pulled)
docker compose up -d

# Verify
docker compose ps
```

### Option 2: Automated Updates (Watchtower)

For fully automated updates, add [Watchtower](https://containrrr.dev/watchtower/) to your compose stack:

```yaml
watchtower:
  image: containrrr/watchtower
  container_name: watchtower
  volumes:
    - /var/run/docker.sock:/var/run/docker.sock
  command: --interval 300  # Check every 5 minutes
  restart: always
```

Add to `compose.yml` and run `docker compose up -d` to auto-update when new images are pushed.

### Option 3: Update on Schedule

Add to `compose.yml`:

```yaml
moodle:
  image: satty008/runmoodle:main
  # Other config...
  restart: always
```

Then on your Proxmox VM, set a cron job:

```bash
# Auto-update every day at 2 AM
0 2 * * * cd /opt/homelab/moodle && docker pull satty008/runmoodle:main && docker compose up -d moodle
```

## Monitoring Builds

### View build logs:
- Go to repo → **Actions** → Click the workflow run

### Get notifications:
- GitHub will email you if builds fail
- Or use Slack/Discord webhooks (optional)

## Updating the Image

When you make changes to the Dockerfile or code:

1. Commit and push to `main`:
   ```bash
   git add moodle/Dockerfile
   git commit -m "feat: update PHP extensions"
   git push origin main
   ```

2. GitHub Actions automatically:
   - Builds the image
   - Pushes to DockerHub
   - Tags it appropriately

3. Your Proxmox VM picks it up (via manual pull or Watchtower)

## Troubleshooting

### Build fails

Check the GitHub Actions logs:
1. Repo → **Actions**
2. Click the failed workflow
3. See the error message
4. Fix and push again

### Image not updating on Proxmox VM

```bash
# Force pull latest
docker pull satty008/runmoodle:main

# Verify the image
docker images satty008/runmoodle

# If using Watchtower, check its logs
docker logs watchtower
```

### Authentication issues

Verify your secrets are set correctly:
1. Go to repo → **Settings** → **Secrets**
2. Confirm `DOCKERHUB_USERNAME` and `DOCKERHUB_TOKEN` exist
3. Re-generate the token if needed

## Benefits of This Approach

✅ **No manual builds** - Git push = automatic image build  
✅ **Version control** - Every commit creates a tagged image  
✅ **Easy rollback** - Any image version can be pulled  
✅ **Scalable** - Deploy to multiple VMs by just pulling  
✅ **Clean workflow** - CI builds, CD deploys  
✅ **Backup** - Images stored in DockerHub  

---

**Next Steps:**
1. Add the secrets to GitHub
2. Push this workflow file
3. Watch the build on GitHub Actions
4. Pull the image on your Proxmox VM
5. Run `docker compose up -d` 🚀
