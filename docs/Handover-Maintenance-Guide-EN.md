# Handover & Maintenance Guide

**Document version:** 1.0  
**Last updated:** 2026-03-02  
**Prepared by:** ATM AB (Agile Transition AB, org nr 559378-3045)  
**Intended audience:** Maintenance organisation taking over responsibility for the four websites

---

## Table of Contents

1. [Introduction & Solution Overview](#1-introduction--solution-overview)
2. [Architecture](#2-architecture)
3. [Tech Stack per Site](#3-tech-stack-per-site)
4. [Deployment Method](#4-deployment-method)
5. [Server Access & Users](#5-server-access--users)
6. [Security](#6-security)
7. [Hosting & Infrastructure](#7-hosting--infrastructure)
8. [Software Upgrades & Maintenance](#8-software-upgrades--maintenance)
9. [Troubleshooting Quick Reference](#9-troubleshooting-quick-reference)
10. [Key Links & Access Points](#10-key-links--access-points)
11. [How to Make Content Changes](#11-how-to-make-content-changes)
12. [Contact Form Backend](#12-contact-form-backend)

---

## 1. Introduction & Solution Overview

**ATM AB (Agile Transition AB, org nr 559378-3045)** designed and built all four websites described in this document. This guide is intended to give a maintenance organisation everything it needs to operate, update, and troubleshoot the solution going forward.

### The four websites

| Site | Domain | GitHub Repository | Type |
|------|--------|-------------------|------|
| Agile Transition | `agiletransition.se` | `Tschiffer46/agiletransition.se` | Static HTML (no build step) |
| AZ Profil | `azprofil.agiletransition.se` | `Tschiffer46/azprofil` | React + Vite static site |
| Padel to Business (azP2B) | `azp2b.agiletransition.se` | `Tschiffer46/azP2B` | React + Vite static site |
| Hemsidor | `hemsidor.agiletransition.se` | `Tschiffer46/hemsidor` | React + Vite static site |

All four sites are **purely static** — there is no server-side application logic and no database. Content is served as pre-built HTML/CSS/JS files.

---

## 2. Architecture

### Traffic flow

```
User → Cloudflare (DNS + CDN + HTTPS) → Hetzner Server (89.167.90.112) → Nginx Proxy Manager → Docker containers (nginx:alpine serving static files)
```

When a visitor opens one of the four websites, the request travels through the following layers:

1. **Cloudflare** resolves the domain name (DNS), caches static assets (CDN), and terminates the HTTPS connection from the user's browser.
2. **Hetzner Server** (IP `89.167.90.112`) receives the request from Cloudflare.
3. **Nginx Proxy Manager** (running as a Docker container on the server) inspects the `Host` header and routes the request to the correct site container.
4. The **site container** (a Docker container running `nginx:alpine`) reads the static files from the `dist/` directory on disk and returns them to the browser.

### Key architectural facts

- All four sites run on **one single Hetzner server** (IP: `89.167.90.112`).
- The server runs **Ubuntu** with **Docker + Docker Compose**.
- All sites are defined in **one master `docker-compose.yml`** at `/home/deploy/hosting/docker-compose.yml`.
- Each site is a separate Docker container running `nginx:alpine`, serving static files from its own `dist/` directory.
- **Nginx Proxy Manager** provides a browser-based GUI at `http://89.167.90.112:81` and handles all traffic routing, SSL certificates (Let's Encrypt), and domain mapping.
- There is **no plain Nginx installed on the host OS** — there are no `/etc/nginx/` configuration files on the server itself.
- All containers share a Docker network named `web`.
- **Cloudflare** handles DNS, CDN caching, and the HTTPS connection from the user to Cloudflare. Let's Encrypt certificates (managed by Nginx Proxy Manager) handle the connection from Cloudflare to the server.

---

## 3. Tech Stack per Site

### agiletransition.se

- Plain HTML — **no build step required**
- Deployed as raw HTML files via `rsync`
- A `CNAME` file exists in the repository (historical GitHub Pages artefact; primary hosting is Hetzner)

### azprofil.agiletransition.se

- React 19 + TypeScript + Vite 7 + Tailwind CSS 3
- `react-i18next` — Swedish and Danish language support
- `react-router-dom` — client-side routing
- Formspree — contact form backend
- **Deploy path on server:** `~/hosting/sites/client-azprofil/dist/`

### azp2b.agiletransition.se (Padel to Business)

- React 19 + TypeScript + Vite 6 + Tailwind CSS 3
- `react-router-dom` — client-side routing
- Swedish only (inline i18n in `src/i18n.ts`)
- **Deploy path on server:** `~/hosting/sites/client-azp2b/dist/`
- > **Note:** This site was originally built with Next.js + Docker + MariaDB, but was simplified to a static Vite site (PR #12). The current version has **no database**.

### hemsidor.agiletransition.se

- React 19 + TypeScript + Vite 6 + Tailwind CSS 3
- Formspree for contact form (form ID: `xreawgqr`)
- Swedish only
- **Deploy path on server:** `~/hosting/sites/hemsidor/dist/`

---

## 4. Deployment Method

### Overview

All four repositories use **GitHub Actions** for automatic deployment. Pushing to the `main` branch triggers the workflow defined in `.github/workflows/deploy.yml` in each repository.

### What the workflow does

1. **Checkout** the repository code.
2. **Setup Node.js 20** (React sites only; skipped for the plain HTML site).
3. **`npm ci`** — install exact dependency versions from `package-lock.json`.
4. **`npm run build`** — compile the React/TypeScript/Vite project into static files in `dist/`.
5. **Setup SSH key** from GitHub Secrets.
6. **`rsync`** — upload the `dist/` folder (or the repo root for `agiletransition.se`) to the Hetzner server.

### Deployment characteristics

- **Zero-downtime deployment**: `rsync` updates files in place while the `nginx:alpine` container continues serving requests.
- **The `--delete` flag** on `rsync` ensures that files removed from the repository are also removed from the server.
- Typical deploy time: **~2 minutes** from push to live.

### Required GitHub Secrets

The following three secrets must be configured in each repository's **Settings → Secrets and variables → Actions**. The values are identical across all four repos.

| Secret | Description |
|--------|-------------|
| `SERVER_HOST` | Hetzner server IP address (`89.167.90.112`) |
| `SERVER_USER` | SSH username (`deploy`) |
| `SERVER_SSH_KEY` | Private SSH key for the `deploy` user |

> ⚠️ **If any of these secrets are missing or incorrect, the deployment will fail.** Check the GitHub Actions log for details.

---

## 5. Server Access & Users

| User | Purpose | How to access |
|------|---------|---------------|
| `root` | Server administration only | SSH via Terminus |
| `deploy` | All deployments — GitHub Actions connects as this user | SSH via Terminus or automatically via GitHub Actions |

- **Recommended SSH client:** Terminus
- The `deploy` user must be a member of the `docker` group. Verify with:

  ```bash
  # Run as root
  groups deploy
  ```

  If `docker` is not listed, add the user:

  ```bash
  # Run as root
  usermod -aG docker deploy
  ```

- **SSH authentication is key-based only** (no password login). The private key stored in the `SERVER_SSH_KEY` GitHub Secret corresponds to a public key in `/home/deploy/.ssh/authorized_keys`.

---

## 6. Security

| Topic | Details |
|-------|---------|
| **SSH access** | Key-based authentication only — password login is disabled |
| **Cloudflare** | Provides DDoS protection, CDN caching, and SSL termination for end users |
| **Let's Encrypt** | SSL certificates between Cloudflare and the server, managed automatically by Nginx Proxy Manager |
| **GitHub Secrets** | SSH keys and server credentials are stored as GitHub Actions secrets — never committed to source code |
| **Docker containers** | Run with `restart: unless-stopped`; static file volumes are mounted read-only (`:ro`) |
| **Nginx Proxy Manager** | "Block Common Exploits" is enabled on all proxy hosts |
| **No database or sensitive data** | All four sites are purely static — no user data is stored on the server |

> ⚠️ **Never commit SSH keys, passwords, or API tokens to any repository.** Always use GitHub Secrets.

---

## 7. Hosting & Infrastructure

| Component | Details |
|-----------|---------|
| **Server provider** | Hetzner (hetzner.com) |
| **Server IP** | `89.167.90.112` |
| **Operating system** | Ubuntu |
| **DNS & CDN** | Cloudflare (dash.cloudflare.com) |
| **Docker** | Docker + Docker Compose installed on the server |
| **Nginx Proxy Manager** | Running as a Docker container; accessible at `http://89.167.90.112:81` |

### Cloudflare DNS configuration

All domains have **A records** pointing to `89.167.90.112`. The **proxy status is enabled** (orange cloud ON) for all domains, meaning all traffic passes through Cloudflare before reaching the server.

### Nginx Proxy Manager

- Accessible at `http://89.167.90.112:81` (browser GUI)
- Each site has a **Proxy Host** entry that maps the domain name to the container name on port 80
- Handles SSL certificate issuance and renewal via Let's Encrypt
- Check this dashboard periodically to confirm that all proxy hosts are active and SSL certificates are valid

---

## 8. Software Upgrades & Maintenance

This section is critical for keeping the solution secure and operational over time.

### What needs upgrading and when

#### 1. Ubuntu OS packages

Security patches should be applied regularly to the server operating system.

```bash
# Run as root
sudo apt update && sudo apt upgrade -y
```

- **Recommended frequency:** Monthly, or immediately when a security advisory is published
- **Optional:** Enable `unattended-upgrades` for automatic security patches:

  ```bash
  # Run as root
  apt install unattended-upgrades
  dpkg-reconfigure --priority=low unattended-upgrades
  ```

#### 2. Docker & Docker Compose

Update when new stable versions are released.

```bash
# Run as root — check current versions
docker --version
docker compose version
```

Follow the [official Docker documentation](https://docs.docker.com/engine/install/ubuntu/) for upgrading on Ubuntu.

#### 3. Nginx Proxy Manager

Update the Nginx Proxy Manager container image.

```bash
# Run as deploy, in /home/deploy/hosting/
docker compose pull proxy-manager
docker compose up -d proxy-manager
```

#### 4. nginx:alpine site containers

The site containers use `nginx:alpine` (latest tag). Pull the latest image and recreate containers:

```bash
# Run as deploy, in /home/deploy/hosting/
docker compose pull
docker compose up -d
```

> ⚠️ Running `docker compose up -d` after a pull will recreate only the containers whose images have changed. Existing containers continue serving traffic until replaced — downtime is minimal.

#### 5. Node.js in GitHub Actions

The GitHub Actions workflows are currently pinned to **Node.js 20**. When Node 20 reaches end-of-life, update the `node-version` field in all four `deploy.yml` files:

```yaml
# .github/workflows/deploy.yml  — change this line
- uses: actions/setup-node@v4
  with:
    node-version: '20'   # Update to next LTS version (e.g. '22')
```

#### 6. npm dependencies in React repositories

Each React repository has `npm` dependencies that may accumulate security vulnerabilities over time.

```bash
# Run locally in each React repo
npm audit
```

- Use `npm update` for minor/patch updates
- Use tools like **Dependabot** or **Renovate** for automated pull requests
- **Major version upgrades** (React, Vite, Tailwind CSS) should be tested locally before merging to `main`

### How to know when upgrades are needed

| Source | What to monitor |
|--------|----------------|
| **GitHub Dependabot** | Enable security alerts and version updates on all four repos: *Settings → Security → Dependabot* |
| **Ubuntu** | Subscribe to [Ubuntu security notices](https://ubuntu.com/security/notices) or enable `unattended-upgrades` |
| **Docker** | Watch [Docker release notes](https://docs.docker.com/engine/release-notes/) |
| **Nginx Proxy Manager** | Check [NPM releases on GitHub](https://github.com/NginxProxyManager/nginx-proxy-manager/releases) |
| **Let's Encrypt certificates** | Auto-renewed by Nginx Proxy Manager — check the NPM dashboard periodically |
| **Hetzner** | Monitor [Hetzner status page](https://www.hetzner.com/status) and email notifications |

### Recommended monthly maintenance checklist

- [ ] SSH into server as `root`, run `sudo apt update && sudo apt upgrade -y`
- [ ] As `deploy`, go to `/home/deploy/hosting/` and run `docker compose pull`
- [ ] Run `docker compose up -d` to apply any updated images
- [ ] Open Nginx Proxy Manager (`http://89.167.90.112:81`) — verify all proxy hosts are green and SSL certificates are valid
- [ ] Check GitHub Actions runs in each repository — verify all recent deploys succeeded
- [ ] Run `npm audit` locally in each React repository to check for vulnerabilities
- [ ] Review any open Dependabot PRs or security alerts on GitHub

---

## 9. Troubleshooting Quick Reference

| Problem | Solution |
|---------|----------|
| **Site shows 502 Bad Gateway** | Container not running — SSH in as `deploy` and run `docker compose ps` in `/home/deploy/hosting/`. Start the container if it is stopped: `docker compose up -d <container-name>` |
| **Site shows old content after deploy** | Restart the container: `docker compose down <container-name> && docker compose up -d <container-name>` |
| **SSL certificate expired** | Open Nginx Proxy Manager → *SSL Certificates* tab → re-request the certificate for the affected domain |
| **GitHub Actions deploy fails** | Check the Actions log in the repository. Verify that all three secrets (`SERVER_HOST`, `SERVER_USER`, `SERVER_SSH_KEY`) are set correctly. Check that the SSH key has not expired or been rotated. |
| **`permission denied` on docker commands** | Run `usermod -aG docker deploy` as root, then log out and back in as `deploy` |
| **Blank page after deploy** | Verify that the `dist/` folder has files: `ls ~/hosting/sites/<site-name>/dist/` |
| **Can't access Nginx Proxy Manager** | Open `http://89.167.90.112:81` in a browser. If unreachable, SSH in and run `docker compose ps` to check whether the `proxy-manager` container is running |

---

## 10. Key Links & Access Points

| Resource | URL / Location |
|----------|---------------|
| **Cloudflare Dashboard** | https://dash.cloudflare.com |
| **Nginx Proxy Manager** | http://89.167.90.112:81 |
| **GitHub repositories** | https://github.com/Tschiffer46 |
| **Hetzner Server IP** | `89.167.90.112` |
| **Docker Compose file** | `/home/deploy/hosting/docker-compose.yml` |
| **Site files — agiletransition** | `~/hosting/sites/client-agiletransition/dist/` |
| **Site files — azprofil** | `~/hosting/sites/client-azprofil/dist/` |
| **Site files — azp2b** | `~/hosting/sites/client-azp2b/dist/` |
| **Site files — hemsidor** | `~/hosting/sites/hemsidor/dist/` |

---

## 11. How to Make Content Changes

### General workflow (all sites)

1. Open the GitHub repository for the site you want to update (https://github.com/Tschiffer46).
2. Edit the relevant file — either via the GitHub web editor or by cloning the repository locally.
3. Commit and push the change to the `main` branch.
4. GitHub Actions automatically builds and deploys within approximately **2 minutes**.
5. Verify the change on the live website.

### agiletransition.se (plain HTML)

Editing `index.html` directly in the repository is sufficient. No build step is needed — the file is uploaded as-is.

### React sites (azprofil, azP2B, hemsidor)

- **Page content** is typically found in component files under `src/components/`
- **Text/translations** for azprofil are in translation files (used by `react-i18next`)
- **Text** for azP2B is in `src/i18n.ts`
- After editing, commit and push to `main` — the GitHub Actions workflow handles the build and deployment automatically

> ⚠️ **Do not push directly to `main` with untested changes on React sites.** Test locally first with `npm run dev`, then push. A broken build will cause the deployment to fail (the live site will not be updated until the next successful push).

---

## 12. Contact Form Backend

| Site | Contact form solution |
|------|-----------------------|
| **azprofil** | Formspree (formspree.io) — form submissions forwarded to configured email |
| **hemsidor** | Formspree (form ID: `xreawgqr`) — form submissions forwarded to configured email |
| **azP2B** | `mailto:` links — no form backend |
| **agiletransition.se** | No contact form |

**Formspree** is a third-party service. The form IDs and associated email addresses are configured in the Formspree dashboard (formspree.io). When taking over responsibility for the sites, ensure that:

1. The Formspree account credentials are transferred to the new responsible party.
2. The email address that receives form submissions is updated to the correct address.
3. The form IDs in the code match the Formspree dashboard (if forms are recreated, the IDs in the source code must be updated and redeployed).
