# Docker Compose Infrastructure

## Overview

This repository contains the complete Docker Compose configuration for my homelab infrastructure.
The architecture is built around **Traefik** as a central reverse proxy with automatic SSL certificate management via Let's Encrypt (OVH DNS challenge).

## Architecture

### Core Infrastructure
- **Traefik** : Reverse proxy, SSL termination, routing
- **Tailscale** : VPN for secure remote access

### Monitoring Stack
- **Prometheus** : Metrics collection and storage
- **Grafana** : Metrics visualization and dashboards
- **Node Exporter** : Host system metrics
- **cAdvisor** : Container metrics
- **Freebox Exporter** : Freebox metrics
- **Unifi Exporter** : Unifi network metrics
- **Restic Exporter** : Backup metrics
- **AdGuard Metrics** : DNS/Ad-blocking metrics

### Applications
- **AdGuard Home** : DNS server and ad-blocking
- **Vaultwarden** : Self-hosted password manager (Bitwarden compatible)
- **n8n** : Workflow automation platform
- **Backrest** : Backup management with restic
- **Trala** : Traefik log analyzer
- **Dockhand** : Docker management interface

## Network Architecture

All services are connected to a shared external network called `proxy`:
```bash
docker network create proxy
```

Traefik routes traffic based on hostnames:
- `traefik.${DOMAIN}` → Traefik dashboard
- `grafana.${DOMAIN}` → Grafana
- `prometheus.${DOMAIN}` → Prometheus
- `vault.${DOMAIN}` → Vaultwarden
- `n8n.${DOMAIN}` → n8n
- `adguard.${DOMAIN}` → AdGuard Home
- `backrest.${DOMAIN}` → Backrest
- `dockhand.${DOMAIN}` → Dockhand

## Setup Instructions

### Prerequisites

1. Docker and Docker Compose installed
2. A domain name configured with OVH DNS
3. OVH API credentials (for Let's Encrypt DNS challenge)

### Initial Setup

1. **Clone this repository**
   ```bash
   git clone <repository-url>
   cd compose
   ```

2. **Create the proxy network**
   ```bash
   docker network create proxy
   ```

3. **Configure OVH secrets for Traefik**
   
   Create the following files in `secrets/traefik/`:
   ```bash
   echo "ovh-eu" > secrets/traefik/ovh_endpoint.secret
   echo "your-application-key" > secrets/traefik/ovh_application_key.secret
   echo "your-application-secret" > secrets/traefik/ovh_application_secret.secret
   echo "your-consumer-key" > secrets/traefik/ovh_consumer_key.secret
   ```

4. **Configure environment variables**
   
   Each service directory contains a `.env.example` file. Copy and customize:
   ```bash
   # For each service directory:
   cd <service-directory>
   cp .env.example .env
   # Edit .env with your values
   ```

   Key variables to configure:
   - `DOMAIN` : Your domain name (in most services)
   - `GRAFANA_ADMIN_USER` and `GRAFANA_ADMIN_PASSWORD` (grafana/)
   - `ADMIN_TOKEN` (vaultwarden/)
   - `N8N_PORT` and `SUBDOMAIN` (n8n/)
   - `UNIFI_USER` and `UNIFI_PASS` (unifi-exporter/)

5. **Prepare Traefik**
   ```bash
   cd traefik
   touch acme.json
   chmod 600 acme.json
   ```

### Deployment Order

1. **Start Traefik first** (reverse proxy must be available)
   ```bash
   cd traefik && docker-compose up -d
   ```

2. **Start monitoring stack**
   ```bash
   cd prometheus && docker-compose up -d
   cd grafana && docker-compose up -d
   cd node-exporter && docker-compose up -d
   cd cadvisor && docker-compose up -d
   ```

3. **Start exporters**
   ```bash
   cd freebox-exporter && docker-compose up -d
   cd unifi-exporter && docker-compose up -d
   cd restic-exporter && docker-compose up -d
   cd adguard-metrics && docker-compose up -d
   ```

4. **Start applications**
   ```bash
   cd adguard && docker-compose up -d
   cd vaultwarden && docker-compose up -d
   cd n8n && docker-compose up -d
   cd backrest && docker-compose up -d
   cd dockhand && docker-compose up -d
   cd trala && docker-compose up -d
   ```

5. **Start Tailscale** (optional, for VPN access)
   ```bash
   cd tailscale && docker-compose up -d
   ```

## SSL Certificates

- SSL certificates are managed automatically by Traefik using Let's Encrypt
- DNS challenge via OVH provider
- Wildcard certificate for `*.${DOMAIN}`
- Certificates stored in `traefik/acme.json`

## Monitoring

Access your monitoring stack:
- **Prometheus**: `https://prometheus.${DOMAIN}`
- **Grafana**: `https://grafana.${DOMAIN}`

Prometheus collects metrics from:
- Traefik (via `:8099/metrics`)
- Node Exporter (host metrics)
- cAdvisor (container metrics)
- Various custom exporters

## Backup Strategy

- **Backrest** provides backup management using restic
- **Restic Exporter** monitors backup health
- Configure backup targets in `backrest/.env`

## Troubleshooting

### Check service logs
```bash
cd <service-directory>
docker-compose logs -f
```

### Verify network connectivity
```bash
docker network inspect proxy
```

### Check Traefik routing
```bash
docker logs traefik
```

### Verify SSL certificates
Check `traefik/acme.json` for certificate entries

## Maintenance

### Update all services
```bash
for dir in */; do
  cd "$dir" && docker-compose pull && docker-compose up -d && cd ..
done
```

### Backup important data
- `traefik/acme.json` (SSL certificates)
- `secrets/` directory
- All `.env` files
- Docker volumes (use Backrest)

## Security Notes

- All secrets are managed via Docker secrets or environment variables
- SSL/TLS enforced on all public services
- Traefik dashboard is password-protected
- AdGuard Home blocks malicious domains
- Vaultwarden admin panel protected by `ADMIN_TOKEN`

## Notes

- Configuration files in the `traefik/custom/` directory provide additional routing rules
- Traefik automatically detects new containers via Docker labels
- HTTP traffic is automatically redirected to HTTPS
