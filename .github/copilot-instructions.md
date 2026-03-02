# Copilot Instructions for agiletransition.se

## Project Overview

This repository contains the **agiletransition.se** website — a plain static HTML site for **Agile Transition AB (ATM AB, org nr 559378-3045)**. It is one of four websites managed under the ATM AB umbrella, all hosted on the same Hetzner server.

For full handover documentation, see [`docs/Handover-Maintenance-Guide-EN.md`](../docs/Handover-Maintenance-Guide-EN.md).

---

## Site Type

- **Plain HTML — no build step.** There is no Node.js, no npm, no React, no Vite.
- The repository root (`index.html` and any supporting assets) is deployed directly via `rsync`.
- Do not suggest adding a build pipeline unless explicitly requested.

---

## Architecture

```
User → Cloudflare (DNS + CDN + HTTPS) → Hetzner Server (89.167.90.112) → Nginx Proxy Manager → Docker container (nginx:alpine)
```

- All four ATM AB sites run on **one Hetzner server** (IP: `89.167.90.112`).
- **Nginx Proxy Manager** (browser GUI at `http://89.167.90.112:81`) handles all routing, SSL, and domain mapping.
- There is **no plain Nginx installed on the host OS** — do not suggest editing `/etc/nginx/` files.
- Each site runs in a Docker container (`nginx:alpine`) on the shared `web` Docker network.
- The master Docker Compose file is at `/home/deploy/hosting/docker-compose.yml` on the server.

---

## Deployment

- Pushing to `main` triggers `.github/workflows/deploy.yml` automatically.
- The workflow uses `rsync` to upload the repository root to the server.
- Typical deploy time: ~2 minutes from push to live.
- **Required GitHub Secrets** (identical across all ATM AB repos):

  | Secret | Value |
  |--------|-------|
  | `SERVER_HOST` | `89.167.90.112` |
  | `SERVER_USER` | `deploy` |
  | `SERVER_SSH_KEY` | Private SSH key for the `deploy` user |

---

## Server Access

- **SSH client used is Terminus** — always refer to it by that name.
- Two server users: `root` (admin only) and `deploy` (deployments and Docker operations).
- SSH authentication is **key-based only** (no password login).
- The `deploy` user must be in the `docker` group.

---

## AI / Copilot Guidelines

- **Do not suggest plain Nginx config files** — all routing is done via Nginx Proxy Manager.
- **Do not suggest adding a build step** (e.g., npm, Vite, webpack) for this repository — it is a plain HTML site.
- **Do not expose container ports directly** (applies when adding new Docker containers to the shared server).
- **Never commit SSH keys, passwords, or API tokens** — always use GitHub Secrets.
- New Docker containers on the shared server must join the `web` network.
- GitHub Secrets for SSH are named `SERVER_HOST`, `SERVER_USER`, `SERVER_SSH_KEY`.
- When referencing the SSH client, always say **Terminus**.
