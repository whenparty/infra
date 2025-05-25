# 🏗️ WhenParty Infrastructure

This repository contains the infrastructure configuration for the **WhenParty** projects. It is designed to manage and deploy services on a **Debian-based VPS** using **Docker**, with **GitHub Actions** for CI/CD and **GitHub Container Registry (GHCR)** for storing Docker images.

## 📁 Folder Structure

```
├── .gitignore               # Git ignore rules
├── docker-compose.yml       # Config for all services
├── README.md                # Project documentation
├── .github/
│   └── workflows/
│       └── copy-files.yaml  # GitHub Actions
├── nginx/
│   ├── certs/               # SSL/TLS certificates (ignored by Git)
│   │   ├── nii.au.crt       # TLS certificate for nii.au
│   │   └── nii.au.key       # Private key for nii.au
│   └── conf.d/
│       └── www.nii.au.conf  # Nginx virtual host configuration
```

## ✨ Key Features

- **Multi-repository design**  
  Each project and infrastructure are maintained in separate repositories.

- **Docker-based deployment**  
  All services run inside Docker containers on a VPS.

- **GitHub Container Registry (GHCR)**  
  CI builds Docker images and pushes them to GHCR for use in production.

- **Automated CI/CD**  
  GitHub Actions handle deployments on push to `main`.

- **Non-root deployments**  
  Services are deployed using a with no root access.

- **Cloudflare & TLS support**  
  Secure HTTPS setup with Cloudflare DNS and valid TLS certificates.

- **Future-ready design**  
  Planned support for WireGuard VPN and additional modular services.

## 🚀 Deployment Workflow

1. **Push to `main`**  
   Commits to `main` (infra or image repos) trigger GitHub Actions.

2. **Build & Push**  
   Docker images are built and pushed to GHCR.

3. **Sync Files via SSH**  
   Updated configuration files are copied to the VPS using `scp`.

4. **Service Restart**  
   Nginx and/or related services are restarted using Docker Compose.

## 🔐 Security & Best Practices

- TLS keys and certs are stored outside Git.
- No builds are performed on the server — all images come from CI.
- SSH-based deploys use a dedicated non-root user.
- Minimal, transparent deployment logic with GitHub Actions.
