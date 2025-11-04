# WhenParty Infrastructure

Nginx reverse proxy infrastructure for the WhenParty project using Cloudflare Origin Certificates.

## Architecture

- **Purpose**: Nginx reverse proxy only (no application services)
- **Networks**: Docker networks `wp_www` (www service) and `wp_tempsites` (temp-site router); additional services (e.g. Telegram bot) should provision their own `wp_*` network
- **TLS**: Cloudflare Origin Certificates with 15-year validity
- **Deployment**: Automated via GitHub Actions

## Prerequisites

### 1. VPS Setup

```bash
# Create Docker networks (run once)
docker network create wp_www
docker network create wp_tempsites

# Create deployment directory
mkdir -p /opt/services/whenparty/infra/nginx/certs
```

### 2. Install Certificates

You must manually install Cloudflare Origin Certificates on the VPS:

1. Generate certificate in Cloudflare Dashboard → SSL/TLS → Origin Server
   - Hostnames: `example.com`, `*.example.com`, `*.temp.example.com`
   - Validity: 15 years
2. Copy certificate and key to VPS:
   ```bash
   # On VPS (replace example.com with your domain)
   nano /opt/services/whenparty/infra/nginx/certs/example.com.crt  # Paste certificate
   nano /opt/services/whenparty/infra/nginx/certs/example.com.key  # Paste private key
   chmod 600 /opt/services/whenparty/infra/nginx/certs/example.com.key
   chmod 644 /opt/services/whenparty/infra/nginx/certs/example.com.crt
   ```

### 3. Cloudflare Settings

```yaml
SSL/TLS:
  Mode: Full (strict) # Critical!
  Edge Certificates:
    Always Use HTTPS: ON

DNS: A    @         YOUR_VPS_IP    Proxied
  A    www       YOUR_VPS_IP    Proxied
```

### 4. GitHub Secrets

Configure these secrets in your GitHub repository:

```yaml
VPS_HOST: your.vps.ip.address
VPS_USER: your-deploy-user
VPS_DEPLOY_KEY: |
  -----BEGIN OPENSSH PRIVATE KEY-----
  [SSH private key for deployment]
  -----END OPENSSH PRIVATE KEY-----
```

## Local Development

Test the nginx configuration locally:

```bash
# Start services
docker compose up -d

# Check logs
docker compose logs -f

# Test health endpoint
curl http://localhost/health

# Reload nginx after config changes
docker exec nginx nginx -s reload
```

## Deployment

Deployment is automated via GitHub Actions:

- **Trigger**: Push to `main` branch with changes to `docker-compose.yml` or `nginx/**`
- **Process**:
  1. Copies `docker-compose.yml` and `nginx/conf.d` to VPS
  2. Runs `docker compose up -d` to update containers
  3. Reloads nginx configuration

## Network Architecture

This infrastructure is part of a multi-repository architecture:

```
 whenparty-infra/          # This repo - Nginx proxy only
 whenparty-www/            # Main website (connects to wp_www)
 whenparty-telegram-bot/   # Telegram bot + PostgreSQL (connects to wp_tg_bot)
 whenparty-temp-sites/     # Temporary subdomains (connects to wp_tempsites)
```

Each service provisions its own external Docker network (`wp_*`). nginx—and any shared infrastructure like monitoring—only joins the networks it must reach (`wp_www`, `wp_tempsites`, future `wp_tg_bot`, etc.), preserving isolation between application containers.

## Nginx Configuration

### Architecture

This infrastructure uses a **service-owned configuration** pattern:

- **Core infrastructure configs** (`nginx/conf.d/default.conf`) - maintained in this repo:

  - HTTP to HTTPS redirection
  - Health check endpoint at `/health`

- **Shared includes** (referenced by service configs):

  - `nginx/conf.d/includes/security-headers.conf` — baseline response headers
  - `nginx/conf.d/cloudflare_real_ip.conf` — Cloudflare IP allow list placeholder (overwritten by automation)

- **Service-specific configs** - maintained in service repos:
  - Each service maintains its own nginx config with all its routing rules
  - Example: `whenparty-www` owns both `www.example.com` and apex redirect (`example.com` → `www.example.com`)
  - During deployment, services copy their config to `nginx/conf.d/`
  - Example: `www.conf` is copied from `whenparty-www` during its deployment

### Service Deployment Pattern

When a service deploys, it follows this pattern:

```bash
# In service repo's .github/workflows/deploy.yml
steps:
  - name: Deploy service and update nginx
    run: |
      # Deploy service
      cd /opt/services/whenparty/www
      docker compose pull
      docker compose up -d

      # Copy nginx config to infra
      cp nginx-config.conf /opt/services/whenparty/infra/nginx/conf.d/www.conf

      # Validate and reload nginx
      docker exec nginx nginx -t && docker exec nginx nginx -s reload
```

This pattern allows:

- Services to own their routing configuration
- Independent service deployments
- Automatic nginx configuration updates
- Clear separation of concerns

### Service Decommissioning

When removing a service:

1. Stop and remove the service container
2. Remove the service's nginx config:
   ```bash
   rm /opt/services/whenparty/infra/nginx/conf.d/service-name.conf
   docker exec nginx nginx -s reload
   ```
3. Each service is responsible for cleaning up its own nginx configuration

**Note**: The infra repo does not automatically remove service configs to avoid accidentally breaking running services.

## Cloudflare Real IP Automation

Use `scripts/update-cloudflare-real-ip.sh` to refresh `/opt/services/whenparty/infra/nginx/conf.d/cloudflare_real_ip.conf` with the latest ranges from Cloudflare. Schedule it with cron so the allow list stays current:

```bash
/opt/services/whenparty/infra/scripts/update-cloudflare-real-ip.sh
```

When the allow list changes the script validates the nginx configuration and reloads the proxy automatically.

### Cron example

Add a cron job for the deploy user:

```bash
crontab -e
0 * * * * /opt/services/whenparty/infra/scripts/update-cloudflare-real-ip.sh >/tmp/update-cloudflare-real-ip.log 2>&1
```

This ensures the allow list is refreshed regularly and nginx reloads when the ranges change.

## Troubleshooting

### Nginx won't start

```bash
# Check nginx config syntax
docker exec nginx nginx -t

# Check logs
docker compose logs nginx
```

### Certificate errors

Ensure certificates exist and have correct permissions:

```bash
ls -la /opt/services/whenparty/infra/nginx/certs/
# Should see example.com.crt (644) and example.com.key (600)
```

### Backend connection errors

Ensure the `www` service is running and connected to `wp_www`:

```bash
docker network inspect wp_www
```

## Key Benefits

- **Simple**: No certificate renewal, no ACME complexity
- **Secure**: Only Cloudflare can reach your VPS
- **Fast**: Cloudflare CDN and DDoS protection
- **Scalable**: Easy to add new services to dedicated `wp_*` networks
- **Automated**: Push to deploy via GitHub Actions

## See Also

- [PROJECT_OVERVIEW.md](./PROJECT_OVERVIEW.md) - Complete architecture documentation
- [Cloudflare Origin Certificates](https://developers.cloudflare.com/ssl/origin-configuration/origin-ca/)
