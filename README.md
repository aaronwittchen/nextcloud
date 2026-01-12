# Nextcloud Kubernetes Deployment

Production-ready Nextcloud deployment for Kubernetes with PostgreSQL and Redis.

## Components

- **Nextcloud 30** - Self-hosted file sync and share
- **PostgreSQL 16** - Database backend with automated backups
- **Redis 7** - Session and file locking cache

## Prerequisites

1. Kubernetes cluster with:
   - Longhorn storage (or local-path for testing)
   - Gateway API with Istio
   - DNS configured for `nextcloud.k8s.local`

2. SOPS and age for secrets encryption
3. ArgoCD with KSOPS plugin (for GitOps deployment)

## Quick Start

### 1. Configure Secrets with SOPS

Secrets are managed with SOPS + age encryption.

```bash
export SOPS_AGE_KEY_FILE=~/age-key.txt

# Edit secrets (replace placeholder passwords)
nano base/secrets.yaml

# Encrypt with SOPS before committing
sops -e -i base/secrets.yaml

# To edit later
sops base/secrets.yaml
```

### 2. Deploy

**Option A: ArgoCD (Recommended)**

```bash
# Commit and push encrypted secrets
git add .
git commit -m "Configure nextcloud secrets"
git push

# Deploy via ArgoCD
kubectl apply -f /path/to/ArgoCD/applications/nextcloud.yaml
```

**Option B: Kustomize**

```bash
# For Longhorn storage (production)
kubectl apply -k overlays/longhorn

# For local-path storage (testing)
kubectl apply -k overlays/local-path
```

### 3. Add DNS Record

Add to your DNS server:
- Domain: `nextcloud.k8s.local`
- IP: Istio Gateway IP

### 4. Access Nextcloud

Open: https://nextcloud.k8s.local

First load may take 1-2 minutes for initial setup.

## Configuration

### Domain Configuration

Currently configured for:

| File                   | Setting                | Value                          |
| ---------------------- | ---------------------- | ------------------------------ |
| `base/deployment.yaml` | NEXTCLOUD_TRUSTED_DOMAINS | `nextcloud.k8s.local`       |
| `base/deployment.yaml` | OVERWRITEHOST          | `nextcloud.k8s.local`          |
| `base/deployment.yaml` | OVERWRITEPROTOCOL      | `https`                        |
| `base/httproute.yaml`  | hostnames              | `nextcloud.k8s.local`          |
| `base/httproute.yaml`  | gateway                | `istio-system/cluster-gateway` |

## File Structure

```
nextcloud/
├── .sops.yaml                # SOPS encryption config
├── base/
│   ├── kustomization.yaml    # Base kustomization
│   ├── namespace.yaml        # Namespace definition
│   ├── service-accounts.yaml # Service accounts
│   ├── secrets.yaml          # SOPS-encrypted secrets
│   ├── ksops-generator.yaml  # KSOPS secret generator
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

## Secrets Setup

Secrets are managed with SOPS encryption using age keys.

### Required Secrets

| Secret | Keys |
|--------|------|
| `postgres-secret` | `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB` |
| `nextcloud-secret` | `NEXTCLOUD_ADMIN_USER`, `NEXTCLOUD_ADMIN_PASSWORD` |

### Encrypt/Decrypt

```bash
export SOPS_AGE_KEY_FILE=~/age-key.txt

# View decrypted content
sops -d base/secrets.yaml

# Edit encrypted file
sops base/secrets.yaml

# Encrypt new file
sops -e -i base/secrets.yaml
```

## Common Operations

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

### Database Connection Issues

```bash
# Test PostgreSQL connectivity
kubectl exec -n nextcloud deployment/nextcloud -- \
  pg_isready -h postgres -p 5432 -U nextcloud
```

### Database Permission Errors

If you see `SQLSTATE[42501]: Insufficient privilege: permission denied for table oc_migrations`:

```bash
# Get postgres credentials
PGUSER=$(kubectl get secret postgres-secret -n nextcloud -o jsonpath='{.data.POSTGRES_USER}' | base64 -d)
PGDB=$(kubectl get secret postgres-secret -n nextcloud -o jsonpath='{.data.POSTGRES_DB}' | base64 -d)

# Fix permissions on existing tables
kubectl exec -it -n nextcloud postgres-0 -- psql -U "$PGUSER" -d "$PGDB" -c \
  "GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO $PGUSER;"
kubectl exec -it -n nextcloud postgres-0 -- psql -U "$PGUSER" -d "$PGDB" -c \
  "GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO $PGUSER;"
kubectl exec -it -n nextcloud postgres-0 -- psql -U "$PGUSER" -d "$PGDB" -c \
  "ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON TABLES TO $PGUSER;"
```

For a fresh install, reset the database entirely:

```bash
# WARNING: This deletes all Nextcloud data!
kubectl exec -it -n nextcloud postgres-0 -- psql -U "$PGUSER" -d postgres -c "DROP DATABASE IF EXISTS $PGDB;"
kubectl exec -it -n nextcloud postgres-0 -- psql -U "$PGUSER" -d postgres -c "CREATE DATABASE $PGDB OWNER $PGUSER;"

# Restart nextcloud pod to trigger reinstall
kubectl delete pod -n nextcloud -l app=nextcloud
```

### Cron Job Failing with "Not installed"

The cron job may fail if Nextcloud hasn't completed initial setup:

```
Exception: Not installed in /var/www/html/lib/base.php
```

**Solution**: Access Nextcloud via browser first to complete the installation wizard. The cron job will work automatically once setup is complete.

## Verification

```bash
# Check all resources
kubectl get all -n nextcloud

# Check pods
kubectl get pods -n nextcloud

# Check HTTPRoute
kubectl get httproute -n nextcloud

# Check Gateway
kubectl get gateway -n istio-system
```

## Security Notes

- Change default admin password immediately after first login
- HTTPS is enabled via Istio Gateway with wildcard certificate
- Apply network policies from `optional/network-policy.yaml`
- Regularly update Nextcloud apps

## Resource Usage

| Component | CPU Request | CPU Limit | Memory Request | Memory Limit |
|-----------|-------------|-----------|----------------|--------------|
| Nextcloud | 200m | 1000m | 512Mi | 1Gi |
| PostgreSQL | 100m | 500m | 256Mi | 512Mi |
| Redis | 50m | 200m | 64Mi | 256Mi |

Total cluster requirements: ~350m CPU, ~832Mi RAM (requests)
