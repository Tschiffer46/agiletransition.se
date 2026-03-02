# agiletransition.se

Static website for **Agile Transition AB** — hosted on Hetzner.

Built with plain HTML, deployed via GitHub Actions (rsync) to a Hetzner server.

## Deployment

Automatic on push to `main` via GitHub Actions → Hetzner (rsync).

Required secrets: `SERVER_HOST`, `SERVER_USER`, `SERVER_SSH_KEY`

## Master Deployment Guide

This repository contains the **master deployment guide** for all ATM AB customer websites:

📖 **[docs/HetznerDeploymentGuide.md](docs/HetznerDeploymentGuide.md)**

This guide covers:
- Server overview and architecture
- How to set up a new customer website (step by step)
- How to switch from preview domain to real domain
- Troubleshooting common issues
- Where to put images in Vite/React repos
- GitHub Actions workflow templates

### Handover & Maintenance Guide

📖 **[docs/Handover-Maintenance-Guide-EN.md](docs/Handover-Maintenance-Guide-EN.md)** (English)
📖 **[docs/Handover-Maintenance-Guide-SV.md](docs/Handover-Maintenance-Guide-SV.md)** (Svenska)

Comprehensive handover documentation covering architecture, security, hosting, deployment, software maintenance, and troubleshooting for all four websites in the solution.