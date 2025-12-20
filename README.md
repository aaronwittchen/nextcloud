# Nextcloud Kubernetes Deployment

Production-ready Nextcloud deployment for Kubernetes with PostgreSQL and Redis.

## Components

- **Nextcloud 30** - Self-hosted file sync and share
- **PostgreSQL 16** - Database backend with automated backups
- **Redis 7** - Session and file locking cache

## Prerequisites

1. Kubernetes cluster with:
   - Longhorn storage (or local-path for testing)
   - Envoy Gateway with Gateway API
   - Pi-hole DNS configured for `nextcloud.k8s.home`

2. Secrets created (see below)

## Quick Start

### 1. Configure Secrets with SOPS

Secrets are managed with SOPS + AGE encryption.

```bash
# Edit secrets (replace placeholder passwords)
nano nextcloud/base/secrets.yaml

# Encrypt with SOPS before committing
sops -e -i nextcloud/base/secrets.yaml

# To edit later
sops nextcloud/base/secrets.yaml
```

**Example unencrypted secrets.yaml:**
```yaml
stringData:
  POSTGRES_PASSWORD: 'your-strong-db-password'
  NEXTCLOUD_ADMIN_PASSWORD: 'your-strong-admin-password'
```

### 2. Deploy with Kustomize

```bash
# For Longhorn storage (production)
kubectl apply -k overlays/longhorn

# For local-path storage (testing)
kubectl apply -k overlays/local-path
```

### 3. Add DNS Record

Add to Pi-hole (192.168.68.10):
- Domain: `nextcloud.k8s.home`
- IP: `192.168.68.200` (MetalLB Ingress IP)

### 4. Access Nextcloud

Open: http://nextcloud.k8s.home

First load may take 1-2 minutes for initial setup.

## File Structure

```
nextcloud/
├── .sops.yaml                # SOPS encryption config
├── base/
│   ├── kustomization.yaml    # Base kustomization
│   ├── namespace.yaml        # Namespace definition
│   ├── service-accounts.yaml # Service accounts
│   ├── secrets.yaml          # SOPS-encrypted secrets
│   ├── pvcs.yaml             # Persistent volume claims
│   ├── postgres-config.yaml  # PostgreSQL configuration
│   ├── postgres.yaml         # PostgreSQL StatefulSet + backup
│   ├── redis.yaml            # Redis deployment
│   ├── deployment.yaml       # Nextcloud deployment + cron
│   └── httproute.yaml        # Gateway API route
├── overlays/
│   ├── longhorn/             # Longhorn storage overlay
│   └── local-path/           # Local-path storage overlay
├── optional/
│   ├── secrets.example.yaml  # Secrets template (reference)
│   ├── network-policy.yaml   # Network policies
│   └── kustomization.yaml    # Optional resources
└── README.md
```

## Storage

| Component | Default Size | Purpose |
|-----------|-------------|---------|
| nextcloud-data-pvc | 50Gi | User files, config, apps |
| postgres-data | 10Gi | Database |
| postgres-backup-pvc | 10Gi | Database backups |

## Automated Tasks

- **Nextcloud Cron**: Runs every 5 minutes for background jobs
- **PostgreSQL Backup**: Daily at 2 AM, 7-day retention

## Configuration

### Increase Upload Limits

Already configured for 16GB uploads. For larger:

```bash
kubectl exec -n nextcloud deployment/nextcloud -- \
  su -s /bin/bash www-data -c "php occ config:system:set upload_max_filesize --value='32G'"
```

### Enable Maintenance Mode

```bash
kubectl exec -n nextcloud deployment/nextcloud -- \
  su -s /bin/bash www-data -c "php occ maintenance:mode --on"
```

### Manual Database Backup

```bash
kubectl exec -n nextcloud postgres-0 -- \
  pg_dump -U nextcloud nextcloud | gzip > nextcloud-backup-$(date +%Y%m%d).sql.gz
```

## Troubleshooting

### Check Pod Status

```bash
kubectl get pods -n nextcloud
kubectl describe pod -n nextcloud <pod-name>
```

### View Logs

```bash
# Nextcloud logs
kubectl logs -n nextcloud deployment/nextcloud

# PostgreSQL logs
kubectl logs -n nextcloud statefulset/postgres

# Redis logs
kubectl logs -n nextcloud deployment/redis
```

### Nextcloud OCC Commands

```bash
kubectl exec -n nextcloud deployment/nextcloud -- \
  su -s /bin/bash www-data -c "php occ <command>"

# Common commands:
# php occ status
# php occ app:list
# php occ files:scan --all
# php occ maintenance:repair
```

### Database Connection Issues

```bash
# Test PostgreSQL connectivity
kubectl exec -n nextcloud deployment/nextcloud -- \
  pg_isready -h postgres -p 5432 -U nextcloud
```

## Security Notes

- Change default admin password immediately after first login
- Consider enabling HTTPS with cert-manager
- Apply network policies from `optional/network-policy.yaml`
- Regularly update Nextcloud apps

## Resource Usage

| Component | CPU Request | CPU Limit | Memory Request | Memory Limit |
|-----------|-------------|-----------|----------------|--------------|
| Nextcloud | 200m | 1000m | 512Mi | 1Gi |
| PostgreSQL | 100m | 500m | 256Mi | 512Mi |
| Redis | 50m | 200m | 64Mi | 256Mi |

Total cluster requirements: ~350m CPU, ~832Mi RAM (requests)
