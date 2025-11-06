# WhenParty Infrastructure - Final Solution with Cloudflare

## Overview

**Architecture**: Multi-repo approach with Cloudflare Origin Certificates (15-year validity)

- **Location**: `/opt/services/whenparty/`
- **Networks**: Docker networks `web_network` (stable services) and `tempsites` (temp-site router)
- **TLS**: Cloudflare Origin Certificates (no renewal needed)
- **Deployment**: GitHub Actions → GHCR → VPS

## Repository Structure

```
whenparty-infra/         # Nginx proxy only
whenparty-www/           # Main website
whenparty-telegram-bot/  # Bot + PostgreSQL (TODO: migrate)
whenparty-temp-sites/    # Temporary subdomains at *.when.party
```

## 1. One-Time VPS Setup

### A. Generate Cloudflare Origin Certificate

1. Go to Cloudflare Dashboard → SSL/TLS → Origin Server
2. Create certificates with hostnames:
   - `nii.au`, `*.nii.au` (stable services)
   - `when.party`, `*.when.party` (temp sites wildcard)
3. Choose 15 years validity
4. Save certificate and private key

### B. Initial VPS Configuration

```bash
#!/bin/bash
# Run once as root on VPS

# Create deployment user
useradd -m -s /bin/bash user
usermod -aG docker user

# Create directory structure
mkdir -p /opt/services/whenparty/{infra,www,telegram-bot,tempsites}
chown -R user:group /opt/services

# Create Docker networks
docker network create web_network
docker network create tempsites
docker network create tempsites

# Install Cloudflare certificates
mkdir -p /opt/services/whenparty/infra/nginx/certs
cd /opt/services/whenparty/infra/nginx/certs

# Stable services certificate (nii.au, *.nii.au)
nano nii.au.crt  # Paste certificate
nano nii.au.key  # Paste private key

# Temp sites certificate (*.when.party)
nano when.party.crt  # Paste certificate
nano when.party.key  # Paste private key

# Set permissions
chmod 600 nii.au.key when.party.key
chmod 644 nii.au.crt when.party.crt
chown -R user:group /opt/services/whenparty
```

### C. Cloudflare Dashboard Settings

```yaml
SSL/TLS:
  Mode: Full (strict) # ← Critical!
  Edge Certificates:
    Always Use HTTPS: ON

DNS: A    @         YOUR_VPS_IP    Proxied ☁️
  A    www       YOUR_VPS_IP    Proxied ☁️
  A    *         YOUR_VPS_IP    Proxied ☁️
```

## 2. Infrastructure Repository

### docker-compose.yml

```yaml
services:
  nginx:
    image: nginx:1.27-alpine
    container_name: nginx
    restart: unless-stopped
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - ./nginx/certs:/etc/nginx/certs:ro
    ports:
      - "80:80"
      - "443:443"
    networks:
      - web_network
    healthcheck:
      test:
        [
          "CMD",
          "wget",
          "--quiet",
          "--tries=1",
          "--spider",
          "http://localhost/health",
        ]
      interval: 30s
      timeout: 3s
      retries: 3

networks:
  web_network:
    external: true
```

### Nginx Configuration Architecture

This infrastructure uses a **service-owned configuration** pattern where each service maintains its own nginx configuration.

#### nginx/conf.d/default.conf (Core Infrastructure)

This file contains only core infrastructure configuration:

```nginx
# HTTP → HTTPS redirect
server {
    listen 80;
    server_name _;

    location = /health {
        return 200 'OK';
        add_header Content-Type text/plain;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}
```

#### Service-Specific Configurations

Service-specific nginx configurations (e.g., `www.conf`) are:

- Maintained in each service's repository
- Copied to `/opt/services/whenparty/infra/nginx/conf.d/` during service deployment
- Automatically loaded by nginx (nginx loads all `.conf` files in the directory)

Example service config structure in `whenparty-www` repo:

```nginx
# www.conf - maintained in whenparty-www repository
upstream www_backend {
    server www:3000;
}

# HTTPS - www.nii.au
server {
    listen 443 ssl http2;
    server_name www.nii.au;

    ssl_certificate /etc/nginx/certs/nii.au.crt;
    ssl_certificate_key /etc/nginx/certs/nii.au.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;

    # Cloudflare real IP
    real_ip_header CF-Connecting-IP;
    real_ip_recursive on;
    include /etc/nginx/conf.d/cloudflare_real_ip.conf;

    # Shared hardening + service-specific security headers
    include /etc/nginx/conf.d/includes/security-headers.conf;
    add_header Content-Security-Policy "default-src 'self';" always;

    location / {
        proxy_pass http://www_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_redirect off;
        proxy_read_timeout 90s;
    }
}

# HTTPS - nii.au → www.nii.au (apex redirect)
server {
    listen 443 ssl http2;
    server_name nii.au;

    ssl_certificate /etc/nginx/certs/nii.au.crt;
    ssl_certificate_key /etc/nginx/certs/nii.au.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;

    return 301 https://www.nii.au$request_uri;
}
```

### .github/workflows/deploy.yml

```yaml
name: Deploy Infrastructure

on:
  push:
    branches: [main]
    paths: ["docker-compose.yml", "nginx/**"]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Copy files to VPS
        uses: appleboy/scp-action@v0.1.5
        with:
          host: ${{ secrets.VPS_HOST }}
          username: ${{ secrets.VPS_USER }}
          key: ${{ secrets.VPS_DEPLOY_KEY }}
          source: "docker-compose.yml,nginx/conf.d"
          target: "/opt/services/whenparty/infra/"

      - name: Deploy
        uses: appleboy/ssh-action@v1.0.0
        with:
          host: ${{ secrets.VPS_HOST }}
          username: ${{ secrets.VPS_USER }}
          key: ${{ secrets.VPS_DEPLOY_KEY }}
          script: |
            set -euo pipefail
            cd /opt/services/whenparty/infra
            docker compose pull
            docker compose up -d
            docker exec nginx nginx -t
            docker exec nginx nginx -s reload
```

## 3. Website Repository

### Dockerfile

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
EXPOSE 3000
CMD ["node", "dist/index.js"]
```

### .github/workflows/deploy.yml

```yaml
name: Deploy Website

on:
  push:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: whenparty/www

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      packages: write
    steps:
      - uses: actions/checkout@v4

      - uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repo
        uses: actions/checkout@v4

      - name: Copy nginx config to VPS
        uses: appleboy/scp-action@v0.1.5
        with:
          host: ${{ secrets.VPS_HOST }}
          username: ${{ secrets.VPS_USER }}
          key: ${{ secrets.VPS_DEPLOY_KEY }}
          source: "nginx-config.conf"
          target: "/tmp/"

      - name: Deploy to VPS
        uses: appleboy/ssh-action@v1.0.0
        env:
          GHCR_TOKEN: ${{ secrets.GHCR_READ_TOKEN }}
        with:
          host: ${{ secrets.VPS_HOST }}
          username: ${{ secrets.VPS_USER }}
          key: ${{ secrets.VPS_DEPLOY_KEY }}
          envs: GHCR_TOKEN
          script: |
            set -euo pipefail
            # Login to GHCR
            echo "$GHCR_TOKEN" | docker login ghcr.io -u whenparty --password-stdin

            # Create service directory
            mkdir -p /opt/services/whenparty/www

            # Create docker-compose
            cat > /opt/services/whenparty/www/docker-compose.yml << 'EOF'
            services:
              www:
                image: ghcr.io/whenparty/www:latest
                container_name: www
                restart: unless-stopped
                networks:
                  - web_network

            networks:
              web_network:
                external: true
            EOF

            # Deploy service
            cd /opt/services/whenparty/www
            docker compose pull
            docker compose up -d

            # Copy nginx config to infra
            cp /tmp/nginx-config.conf /opt/services/whenparty/infra/nginx/conf.d/www.conf

            # Validate and reload nginx
            docker exec nginx nginx -t && docker exec nginx nginx -s reload
```

## 4. Service Deployment Flow

### How Services Deploy and Register with Nginx

Each service follows this pattern during deployment:

1. **Build**: Service builds Docker image and pushes to GHCR
2. **Deploy Service**: Service deploys its container to `/opt/services/whenparty/<service-name>/`
3. **Register with Nginx**: Service copies its nginx config to infra's `nginx/conf.d/` directory
4. **Reload**: Nginx configuration is validated and reloaded

This architecture provides:

- **Service Autonomy**: Each service owns its routing configuration
- **Independent Deployments**: Services can deploy without modifying infra repo
- **Automatic Registration**: Services "register" themselves by copying their config
- **Clear Separation**: Core infrastructure vs. service-specific routing

### Directory Structure on VPS

```
/opt/services/whenparty/
├── infra/                          # Infrastructure repo
│   ├── docker-compose.yml          # Nginx container definition
│   ├── nginx/
│   │   ├── certs/
│   │   │   ├── nii.au.crt         # Cloudflare Origin cert for nii.au, *.nii.au
│   │   │   ├── nii.au.key
│   │   │   ├── when.party.crt    # Cloudflare Origin cert for *.when.party
│   │   │   └── when.party.key
│   │   └── conf.d/
│   │       ├── default.conf       # Core: HTTP redirect, health check
│   │       ├── www.conf           # Copied from whenparty-www (includes apex redirect)
│   │       └── temp-sites.conf    # Wildcard routing for *.when.party → Traefik
│   └── .github/workflows/deploy.yml
│
├── www/                            # Website service (stable)
│   ├── docker-compose.yml          # www container (connects to web_network)
│   └── nginx-config.conf           # Service's nginx config (source)
│
├── telegram-bot/                   # Bot service (stable)
│   └── docker-compose.yml
│
└── tempsites/                      # Temp sites (image-based deployment)
    └── docker-compose.yml          # Auto-generated: Traefik + all projects (from GHCR images)
                                    # No git, no local files - just pulls images
```

## 5. Temp Sites Architecture (`*.when.party`)

The temp-sites system provides a clean, low-friction way to deploy disposable web applications at `*.when.party` using **image-based deployment** with Traefik-based dynamic routing.

### Architecture Overview

```
Internet → Cloudflare → Nginx (TLS) → Traefik (HTTP) → Project Containers (GHCR Images)
                        :443           :8000             (various ports)
```

**Key Design Decisions:**

1. **Image-Based Deployment**: No git on VPS - everything deployed as Docker images from GHCR (public)
2. **Separate Certificate**: Uses dedicated Cloudflare Origin cert for `*.when.party` (separate from `nii.au`)
3. **Isolated Network**: Temp sites use `tempsites` Docker network (isolated from `web_network`)
4. **Traefik Routing**: Dynamic routing via Docker labels - no nginx config churn
5. **TLS at Nginx**: Nginx terminates TLS; Traefik is HTTP-only (simpler, no ACME complexity)
6. **Auto-Generated Compose**: Single `docker-compose.yml` generated from project structure
7. **Separate Repository**: `whenparty-temp-sites` repo keeps disposable projects isolated from stable services

### How It Works

1. **Nginx Wildcard Config** (`temp-sites.conf`):

   - Catches all requests to `*.when.party`
   - Terminates TLS with `when.party.crt`
   - Proxies to Traefik on `localhost:8000`

2. **Traefik Router**:

   - Runs from generated `docker-compose.yml` on VPS
   - HTTP-only on port 8000 (localhost)
   - Docker provider discovers project containers via labels
   - Routes by Host header (`project-name.when.party`)

3. **Project Images** (GitHub Actions):

   - Each project in `projects/<name>/` must have a `Dockerfile`
   - GitHub Actions builds images for changed projects
   - Images pushed to GHCR as public packages: `ghcr.io/org/temp-<project>:latest`
   - Generates complete `docker-compose.yml` with Traefik + all projects
   - SCPs compose file to VPS

4. **VPS Deployment**:
   - VPS has `/opt/services/whenparty/tempsites/docker-compose.yml` only (no git)
   - Runs `docker compose pull && up -d` to pull images and start containers
   - All containers connect to `tempsites` network
   - Completely isolated from other projects and stable services

### Adding a New Temp Project

**Static Site Example:**

```bash
# 1. Create project structure
mkdir -p projects/my-site/site

# 2. Create Dockerfile
cat > projects/my-site/Dockerfile << 'EOF'
FROM nginx:alpine
COPY site/ /usr/share/nginx/html/
EXPOSE 80
EOF

# 3. Add site content
cat > projects/my-site/site/index.html << 'EOF'
<!DOCTYPE html>
<html>
<head><title>My Site</title></head>
<body><h1>Hello from my-site.when.party</h1></body>
</html>
EOF

# 4. Commit and push - GitHub Actions builds image and deploys
git add projects/my-site
git commit -m "Add my-site temp site"
git push origin main
```

**What happens:**

1. GitHub Actions detects `projects/my-site/` changed
2. Builds Docker image from Dockerfile
3. Pushes to `ghcr.io/org/temp-my-site:latest` (public)
4. Generates `docker-compose.yml` with all projects (including my-site)
5. SCPs compose file to VPS
6. VPS runs `docker compose pull && up -d`
7. Traefik routes `my-site.when.party` → container
8. Site is live!

**Custom Application Example:**

```bash
# Node.js app with custom port
mkdir -p projects/my-app/src

cat > projects/my-app/Dockerfile << 'EOF'
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY src/ ./src/
EXPOSE 3000
CMD ["node", "src/index.js"]
EOF

# Optional: Configure port in .project-config
cat > projects/my-app/.project-config << 'EOF'
PROJECT_PORT=3000
PROJECT_SUBDOMAIN=my-app
EOF

git add projects/my-app
git commit -m "Add my-app"
git push origin main
```

### Deployment Flow

When you push changes to `projects/`:

1. **Change Detection**: GitHub Actions detects which projects changed
2. **Build Images**: Builds Docker images for changed projects (parallel matrix)
3. **Push to GHCR**: Pushes images as public packages (`ghcr.io/org/temp-project:latest`)
4. **Generate Compose**: Runs `scripts/generate-compose.sh` to create complete `docker-compose.yml`
5. **Deploy to VPS**: SCPs compose file to VPS, then SSHes and runs `docker compose pull && up -d`
6. **Routing**: Traefik automatically discovers containers via labels and routes traffic

**Key Point**: VPS never uses git - it only pulls Docker images and runs the compose file.

### Lifecycle Management

- **Update Project**: Edit files, commit, push - GitHub Actions rebuilds and redeploys
- **Remove Project**: Delete directory from repo, commit, push - next deployment won't include it
- **Manual Control on VPS**:
  - Stop: `docker compose stop my-project`
  - Restart: `docker compose restart my-project`
  - Remove: Delete from repo and redeploy, or manually edit VPS compose file

### Benefits vs. Per-Service Nginx Configs

| Traditional (nginx per-service) | Temp Sites (Traefik)            |
| ------------------------------- | ------------------------------- |
| Edit nginx config per project   | Add Traefik labels in compose   |
| Commit to infra repo            | Commit to temp-sites repo       |
| Reload nginx                    | Automatic discovery             |
| 1 file per service              | 1 label block per service       |
| Good for stable services        | Perfect for disposable projects |

### Network Isolation

```
web_network (stable services)     tempsites (temp projects)
├── nginx ←────┐                 ├── traefik-tempsites
├── www        │                 ├── dtm-cleaning
└── telegram-bot                 └── my-project
               │
               └─ Nginx listens on both networks via ports
```

- Stable services (`www`, `telegram-bot`) cannot communicate with temp projects
- Temp projects cannot communicate with each other or stable services
- Only nginx bridges the networks via HTTP proxying

### Traefik Labels Reference

**Basic routing:**

```yaml
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.NAME.rule=Host(`subdomain.when.party`)"
  - "traefik.http.routers.NAME.entrypoints=web"
  - "traefik.http.services.NAME.loadbalancer.server.port=PORT"
```

**Multiple domains:**

```yaml
labels:
  - "traefik.http.routers.NAME.rule=Host(`app1.when.party`) || Host(`app2.when.party`)"
```

**Path-based routing:**

```yaml
labels:
  - "traefik.http.routers.NAME.rule=Host(`app.when.party`) && PathPrefix(`/api`)"
```

### Troubleshooting

1. **Site not accessible:**

   ```bash
   # On VPS
   cd /opt/services/whenparty/tempsites

   # Check compose file exists
   cat docker-compose.yml

   # Check containers running
   docker compose ps

   # Check Traefik logs
   docker logs traefik-tempsites

   # Check project logs
   docker compose logs my-project

   # Test Traefik routing locally
   curl -H "Host: my-project.when.party" http://localhost:8000
   ```

2. **Image not found:**

   ```bash
   # Verify image exists on GHCR
   # Visit: https://github.com/orgs/<org>/packages?repo_name=temp-sites

   # Try pulling manually
   docker pull ghcr.io/<org>/temp-my-project:latest

   # Check if image is public (if auth required, make it public in GitHub UI)
   ```

3. **Certificate errors:**

   ```bash
   # Verify cert covers *.when.party
   openssl x509 -in /opt/services/whenparty/infra/nginx/certs/when.party.crt -text | grep DNS
   ```

4. **Routing not working:**

   ```bash
   # Check Traefik discovered container
   docker logs traefik-tempsites | grep my-project

   # Verify container is on tempsites network
   docker network inspect tempsites

   # Check container labels
   docker inspect my-project | grep traefik
   ```

For complete documentation, see: `whenparty-temp-sites/README.md`

## 6. GitHub Secrets Required

```yaml
VPS_HOST: your.vps.ip
VPS_USER: user
VPS_DEPLOY_KEY: |
  -----BEGIN OPENSSH PRIVATE KEY-----
  [your SSH private key]
  -----END OPENSSH PRIVATE KEY-----
GHCR_READ_TOKEN: ghp_xxxx # Personal Access Token with read:packages
```

## 6. Migration from Current Setup

```bash
#!/bin/bash
# Run on VPS to migrate from current setup

# Stop current services
cd /home/user/projects/infra
docker compose down

# Move certificates
cp -r nginx/certs /opt/services/whenparty/infra/nginx/

# Remove www service from old docker-compose
# Then push updated repos - they'll auto-deploy to new location
```

## Key Benefits of This Solution

✅ **Simple**: No certificate renewal, no ACME complexity  
✅ **Secure**: Only Cloudflare can reach your VPS  
✅ **Fast**: Cloudflare CDN and DDoS protection  
✅ **Scalable**: Easy to add new services  
✅ **Automated**: Push to deploy via GitHub Actions  
✅ **15-year certs**: Set once, forget

## TODO

- Migrate telegram bot to web_network (currently on port 8443)
- Consider adding monitoring (Uptime Kuma)
