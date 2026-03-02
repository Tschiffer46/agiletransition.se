# ATM AB — Hetzner Server Deployment Guide

This document describes how the Hetzner web hosting server is set up and how to deploy new customer websites onto it. It is written for all future projects and should be kept up to date.

---

## Server Overview

| Detail | Value |
|---|---|
| Provider | Hetzner |
| Server IP | 89.167.90.112 |
| SSH client | Terminus |
| Web server | Nginx Proxy Manager (browser-based GUI) |
| Container tool | Docker + Docker Compose |
| Deployment | GitHub Actions (automatic on push to `main`) |

The server runs **all websites together** from one single file:
```
/home/deploy/hosting/docker-compose.yml
```

There is no plain Nginx installed. All traffic routing is handled by **Nginx Proxy Manager** — a browser-based tool accessible at:
```
http://89.167.90.112:81
```

---

## Server Users

| User | Purpose |
|---|---|
| `root` | Server administration only |
| `deploy` | All deployments — GitHub Actions connects as this user |

The `deploy` user must be in the `docker` group. To verify, log in as `root` in Terminus and run:
```bash
groups deploy
```
You should see `docker` in the list. If not, run:
```bash
usermod -aG docker deploy
```
Then verify again with `groups deploy`.

---

## How the Server Works

Every website on the server is a **Docker container** defined in the master `docker-compose.yml` file. There are two types of sites:

### Type A — Static sites (simple HTML/React/Vite)
These are the simplest. GitHub Actions builds the site and copies the files to the server via `rsync`. The container just serves the files.

Example entry in `docker-compose.yml`:
```yaml
  sitename:
    image: nginx:alpine
    container_name: sitename
    restart: unless-stopped
    volumes:
      - ./sites/client-sitename/dist:/usr/share/nginx/html:ro
    networks:
      - web
```

Required GitHub Secrets:
| Secret | Value |
|---|---|
| `SERVER_HOST` | 89.167.90.112 |
| `SERVER_USER` | deploy |
| `SERVER_SSH_KEY` | Private SSH key for the deploy user |

### Type B — Full-stack apps (Next.js + database)

> **⚠️ Note:** Type B deployments are significantly more complex and have more failure modes. For simple marketing/company websites, always prefer Type A (static site with rsync). Only use Type B if the site genuinely requires a server-side database.

These are more complex. GitHub Actions builds a Docker image, pushes it to the GitHub Container Registry (GHCR), then SSHes into the server and updates the master `docker-compose.yml` to run the new app and its database.

Required GitHub Secrets:
| Secret | Value |
|---|---|
| `SERVER_HOST` | 89.167.90.112 |
| `SERVER_USER` | deploy |
| `SERVER_SSH_KEY` | Private SSH key for the deploy user |
| `GHCR_TOKEN` | GitHub Personal Access Token with `write:packages` scope — **alternatively, use the automatic `GITHUB_TOKEN` (preferred, no manual PAT needed); the workflow just needs `permissions: packages: write`** |
| `DB_PASSWORD` | MariaDB user password |
| `DB_ROOT_PASSWORD` | MariaDB root password |
| `ADMIN_PASSWORD` | Admin panel password |
| `SMTP_HOST` | Email server hostname |
| `SMTP_PORT` | Email server port (usually 587) |
| `SMTP_USER` | Email username |
| `SMTP_PASS` | Email password |
| `SMTP_FROM` | From address for outgoing emails |

---

## Setting Up a New Customer Website

### Step 1 — Create the GitHub repository
Create a new repository under `Tschiffer46` named after the customer (e.g. `client-customername`).

Add all required GitHub Secrets under **Settings → Secrets and variables → Actions**.

### Step 2 — Add a subdomain for customer preview
While waiting to receive access to the customer's own domain, use a subdomain under `agiletransition.se` for previewing the site.

1. Go to [dash.cloudflare.com](https://dash.cloudflare.com) and log in
2. Click on **agiletransition.se**
3. Click **DNS** in the left menu
4. Click **Add record** and fill in:

| Field | Value |
|---|---|
| Type | `A` |
| Name | `customername` (e.g. `azp2b`) |
| IPv4 address | `89.167.90.112` |
| Proxy status | Proxied (orange cloud ON) |

5. Click **Save**

The preview URL will be: `https://customername.agiletransition.se`

### Step 3 — Add a Proxy Host in Nginx Proxy Manager
1. Open a browser and go to `http://89.167.90.112:81`
2. Log in
3. Click **Add Proxy Host** (green button, top right)
4. Fill in the **Details** tab:

| Field | Value |
|---|---|
| Domain Names | `customername.agiletransition.se` |
| Scheme | `http` |
| Forward Hostname / IP | Container name (e.g. `azp2b`) |
| Forward Port | `80` for static sites, `3000` for Next.js apps |
| Block Common Exploits | On |

5. Click the **SSL** tab:

| Field | Value |
|---|---|
| SSL Certificate | Request a new SSL Certificate |
| Force SSL | On |
| HTTP/2 Support | On |
| Email address | Your email address |
| I Agree to Let's Encrypt ToS | ✅ Ticked |

6. Click **Save**

### Step 4 — Add the site to the master docker-compose.yml
Log in to the server as `deploy` in Terminus and edit the master compose file:
```bash
nano /home/deploy/hosting/docker-compose.yml
```

**For a static site (Type A)**, add:
```yaml
  customername:
    image: nginx:alpine
    container_name: customername
    restart: unless-stopped
    volumes:
      - ./sites/client-customername/dist:/usr/share/nginx/html:ro
    networks:
      - web
```

**For a full-stack app (Type B)**, add:
```yaml
  customername:
    image: ghcr.io/tschiffer46/customername:latest
    container_name: customername
    restart: unless-stopped
    env_file:
      - .env.customername
    depends_on:
      customername-db:
        condition: service_healthy
    networks:
      - web

  customername-db:
    image: mariadb:11
    container_name: customername-db
    restart: unless-stopped
    env_file:
      - .env.customername
    environment:
      MARIADB_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
      MARIADB_DATABASE: customername
      MARIADB_USER: customername
      MARIADB_PASSWORD: ${DB_PASSWORD}
    volumes:
      - customername_db_data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "healthcheck.sh", "--connect", "--innodb_initialized"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 120s
    networks:
      - web
```

And add to the `volumes:` section at the bottom:
```yaml
  customername_db_data:
```

Then restart only the new containers (this does NOT affect other running sites).

**Important:** Before running `docker compose up`, make sure the target directory exists and is owned by `deploy`:
```bash
mkdir -p /home/deploy/hosting/sites/client-customername/dist
chown -R deploy:deploy /home/deploy/hosting/sites/client-customername
```

Then start the container:
```bash
cd /home/deploy/hosting
docker compose up -d customername
```

### Step 5 — Deploy the site
Push to `main` in the GitHub repository. GitHub Actions will build and deploy automatically.

---

## Switching from Preview Domain to Real Domain

When you receive access to the customer's real domain (e.g. `customername.se`):

1. **Update DNS** — In Cloudflare, add an A record for `customername.se` pointing to `89.167.90.112`
2. **Temporarily turn OFF the Cloudflare proxy** (grey cloud) so Let's Encrypt can issue a certificate
3. **Add a new Proxy Host** in Nginx Proxy Manager for `customername.se` pointing to the same container
4. **Get the SSL certificate** in the new proxy host
5. **Turn the Cloudflare proxy back ON** (orange cloud)
6. **Delete or update the old preview proxy host** (`customername.agiletransition.se`) if no longer needed

---

## Where to Put Images (Vite/React Sites)

For Vite/React sites (Type A), images go in the `public/` folder. Everything in `public/` is copied as-is to the build output (`dist/`).

### Recommended folder structure

```
public/
├── favicon.ico
└── assets/
    ├── hero/
    │   └── hero.jpg              (1920×1080px min, background image)
    ├── logo/
    │   ├── logo.png              (200×60px, white/transparent, for dark backgrounds)
    │   └── logo-dark.png         (200×60px, dark version, optional)
    ├── team/
    │   ├── founders.jpg          (1000×667px, group photo)
    │   ├── person-name.jpg       (400×400px, square portrait)
    │   └── ...
    ├── og/
    │   └── og-image.jpg          (1200×630px exactly, for social sharing)
    └── partners/                  (if applicable)
        ├── partner-name.png      (PNG, transparent background)
        └── ...
```

### How to add images

1. Add the image files to the correct folder in the repo
2. Commit and push to `main`
3. GitHub Actions will build and deploy automatically — images are included in the `dist/` output

### How to reference images in code

In React components, reference images in `public/` using absolute paths from the root:
```tsx
<img src="/assets/hero/hero.jpg" alt="Hero" />
<img src="/assets/logo/logo.png" alt="Logo" />
```

### Image guidelines

- Use `.jpg` or `.webp` for photos (keep under 500KB each)
- Use `.png` with transparent background for logos
- Always optimize images before committing (use tinypng.com or similar)
- Each image folder should contain a `README.md` listing required files and sizes

---

## GitHub Actions Workflow Template

### For static sites (Type A) — with build step (Vite/React)

Create `.github/workflows/deploy.yml`:

```yaml
# Required GitHub Secrets (same names used across all ATM AB deployments):
#   SERVER_HOST     - IP address or hostname of the Hetzner server
#   SERVER_USER     - SSH username (deploy)
#   SERVER_SSH_KEY  - Private SSH key content for authentication
#
# To add secrets: GitHub repo → Settings → Secrets and variables → Actions → New repository secret

name: Deploy to Hetzner

on:
  push:
    branches:
      - main

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Build
        run: npm run build

      - name: Setup SSH
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.SERVER_SSH_KEY }}" > ~/.ssh/id_ed25519
          chmod 600 ~/.ssh/id_ed25519
          ssh-keyscan -H ${{ secrets.SERVER_HOST }} > ~/.ssh/known_hosts

      - name: Deploy to server
        run: |
          rsync -avz --delete \
            --exclude='.git' \
            --exclude='.github' \
            --exclude='README.md' \
            --exclude='node_modules' \
            -e "ssh -i ~/.ssh/id_ed25519" \
            ./dist/ ${{ secrets.SERVER_USER }}@${{ secrets.SERVER_HOST }}:~/hosting/sites/client-CUSTOMERNAME/dist/
```

Replace `CUSTOMERNAME` with the container name used in `docker-compose.yml`.

### For static sites (Type A) — plain HTML (no build step)

```yaml
name: Deploy to Hetzner

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup SSH
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.SERVER_SSH_KEY }}" > ~/.ssh/id_ed25519
          chmod 600 ~/.ssh/id_ed25519
          ssh-keyscan -H ${{ secrets.SERVER_HOST }} > ~/.ssh/known_hosts

      - name: Deploy to server
        run: |
          rsync -avz --delete \
            --exclude='.git' \
            --exclude='.github' \
            --exclude='README.md' \
            --exclude='CNAME' \
            -e "ssh -i ~/.ssh/id_ed25519" \
            ./ ${{ secrets.SERVER_USER }}@${{ secrets.SERVER_HOST }}:~/hosting/sites/client-CUSTOMERNAME/dist/
```

---

## Verifying a Deployment

After deploying, check the following in Terminus (log in as `deploy`):

**Check all containers are running:**
```bash
docker compose -f /home/deploy/hosting/docker-compose.yml ps
```

**Check logs for a specific container:**
```bash
docker compose -f /home/deploy/hosting/docker-compose.yml logs --tail=50 containername
```

**Check the site in a browser:**
Open `https://customername.agiletransition.se` — it should load correctly.

---

## Troubleshooting

| Problem | Solution |
|---|---|
| `permission denied` when running docker commands as deploy | Run `usermod -aG docker deploy` as root, then open a new deploy session in Terminus |
| Database container keeps restarting | Wait 30 seconds for the healthcheck, then check logs: `docker compose logs p2b-db` |
| App container not starting | Check logs for database connection errors: `docker compose logs containername` |
| 502 Bad Gateway in browser | Container is not running — check `docker compose ps` |
| SSL certificate fails | Make sure DNS is pointing to `89.167.90.112` and Cloudflare proxy is OFF (grey cloud) during certificate issuance |
| Site shows blank page | Check that the `dist/` folder has files in it — it may be empty if the first deploy hasn't run yet |
| Nginx Proxy Manager shows wrong port | Edit the proxy host and update the Forward Port — `80` for static, `3000` for Next.js |
| `sudo: a password is required` during deploy | The deploy workflow should NOT use `sudo`. If rsync fails with permission denied, check that the target directory exists and is owned by `deploy` — run `chown -R deploy:deploy ~/hosting/sites/client-X` as root |
| rsync `Permission denied` on first deploy | The target directory doesn't exist yet. As root, run: `mkdir -p /home/deploy/hosting/sites/client-X/dist && chown -R deploy:deploy /home/deploy/hosting/sites/client-X` |
| Docker compose file corrupted by automated script | Never let GitHub Actions modify docker-compose.yml via scripts. Always edit it manually on the server. Use Type A (static rsync) deployments. |
| Container running but site shows old/wrong content | The container may be using a cached image. Run: `cd /home/deploy/hosting && docker compose down containername && docker compose up -d containername` |

---

## Master docker-compose.yml Reference

The master file lives at `/home/deploy/hosting/docker-compose.yml`. Always use this file for all `docker compose` commands:
```bash
docker compose -f /home/deploy/hosting/docker-compose.yml [command]
```

Never run `docker compose` from any other directory unless you know what you are doing — it may create a separate isolated stack instead of updating the shared one.

---

## Notes for Copilot / AI Assistants

When helping with this project, always follow these rules:

- The SSH client used is **Terminus** — always refer to it by that name
- There is **no plain Nginx** installed — do not suggest editing Nginx config files
- There are **no** `/etc/nginx/conf.d/` or `sites-available` directories
- All routing is done via **Nginx Proxy Manager** at `http://89.167.90.112:81`
- The master `docker-compose.yml` is at `/home/deploy/hosting/docker-compose.yml`
- All sites share the `web` Docker network
- New containers must join the `web` network
- Type B app containers must **not** expose ports directly — Nginx Proxy Manager reaches them via the Docker network
- GitHub Secrets for SSH are named `SERVER_HOST`, `SERVER_USER`, `SERVER_SSH_KEY`
- The server runs Ubuntu on Hetzner
