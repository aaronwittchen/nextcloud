# Nextcloud Deployment Guide

Complete guide from code to production deployment using GitOps with ArgoCD and SOPS encryption.

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Repository Setup](#repository-setup)
3. [SOPS Encryption](#sops-encryption)
4. [DNS Configuration](#dns-configuration)
5. [ArgoCD Deployment](#argocd-deployment)
6. [Verification](#verification)
7. [Post-Installation](#post-installation)
8. [Troubleshooting](#troubleshooting)

---

## Prerequisites

### Cluster Requirements

| Component     | Version | Purpose                   |
| ------------- | ------- | ------------------------- |
| Kubernetes    | 1.31+   | Orchestration             |
| Longhorn      | 1.7+    | Persistent storage        |
| Envoy Gateway | -       | Ingress routing           |
| MetalLB       | 0.14+   | LoadBalancer IPs          |
| ArgoCD        | 2.x     | GitOps deployment         |
| KSOPS         | -       | SOPS decryption in ArgoCD |

### Local Tools

```bash
# Required on your local machine
sudo dnf install -y epel-release
sudo dnf install -y age

# Download the latest sops binary
SOPS_VERSION="3.11.0"  # Check for latest at github.com/getsops/sops/releases
curl -LO https://github.com/getsops/sops/releases/download/v${SOPS_VERSION}/sops-v${SOPS_VERSION}.linux.amd64

# Make it executable and move to PATH
chmod +x sops-v${SOPS_VERSION}.linux.amd64
sudo mv sops-v${SOPS_VERSION}.linux.amd64 /usr/local/bin/sops

# Verify installation
sops --version
age --version
```

### Verify Cluster Access

```bash
# From your local machine
kubectl get nodes
kubectl get pods -n argocd
```

---

## Repository Setup

### 1. Clone/Fork Repository

```bash
git clone https://github.com/aaronwittchen/nextcloud.git
cd nextcloud
```

## SOPS Encryption

### 1. Locate Your AGE Key

# Create the directory

mkdir -p ~/.config/sops/age

# Generate a new age key pair

age-keygen -o ~/.config/sops/age/keys.txt

# Secure the permissions

chmod 600 ~/.config/sops/age/keys.txt

# View your NEW public key

age-keygen -y ~/.config/sops/age/keys.txt

```

Your AGE public key (from `.sops.yaml`):

```

age14c4u5y4uf869su746ghhear0ys4p6fuj6r0pvzrujj03dwh3xals40k30e

````

Your private key should be stored securely (e.g., `~/.config/sops/age/keys.txt`).

### 2. Configure SOPS Environment

```bash
# Set AGE key location
export SOPS_AGE_KEY_FILE=~/.config/sops/age/keys.txt

# Verify SOPS can find the key
sops --version
````

### 3. Edit Secrets

```bash
# Open secrets file
nano base/secrets.yaml
```

### 4. Encrypt Secrets

```bash
# Encrypt in-place
sops -e -i base/secrets.yaml

# Verify encryption (should show ENC[AES256_GCM,...])
cat base/secrets.yaml
```

**Expected output after encryption:**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
  namespace: nextcloud
type: Opaque
stringData:
  POSTGRES_USER: ENC[AES256_GCM,data:...,type:str]
  POSTGRES_PASSWORD: ENC[AES256_GCM,data:...,type:str]
  POSTGRES_DB: ENC[AES256_GCM,data:...,type:str]
sops:
  age:
    - recipient: age14c4u5y4uf869su746ghhear0ys4p6fuj6r0pvzrujj03dwh3xals40k30e
      enc: |
        -----BEGIN AGE ENCRYPTED FILE-----
        ...
```

### 5. Test Decryption (Optional)

```bash
# Decrypt to stdout (doesn't modify file)
sops -d base/secrets.yaml
```

### 6. Commit Encrypted Secrets

```bash
git add nextcloud/
git commit -m "Add Nextcloud with encrypted secrets"
git push origin main
```

---

## DNS Configuration

### Add Pi-hole DNS Record

**Option A: Web Interface**

1. Open `http://192.168.68.10/admin`
2. Navigate to **Local DNS** → **DNS Records**
3. Add entry:
   - Domain: `nextcloud.k8s.home`
   - IP: `192.168.68.200`
4. Click **Add**

**Option B: Wildcard (if configured)**

If you have wildcard DNS (`*.k8s.home → 192.168.68.200`), no action needed.

### Verify DNS

```bash
# From your local machine
nslookup nextcloud.k8s.home
# Should return: 192.168.68.200

# Or using dig
dig nextcloud.k8s.home @192.168.68.10
```

---

## ArgoCD Deployment

### 1. Update ArgoCD Application

Edit `ArgoCD/applications/nextcloud.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nextcloud
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default

  source:
    repoURL: https://github.com/aaronwittchen/nextcloud.git
    targetRevision: master
    path: overlays/longhorn

  destination:
    server: https://kubernetes.default.svc
    namespace: nextcloud

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### 2. Apply ArgoCD Application

**Option A: Via kubectl**

```bash
kubectl apply -f applications/nextcloud.yaml
```

**Option B: Via ArgoCD CLI**

```bash
argocd app create nextcloud \
  --repo https://github.com/YOUR_USERNAME/YOUR_REPO.git \
  --path nextcloud/overlays/longhorn \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace nextcloud \
  --sync-policy automated \
  --auto-prune \
  --self-heal
```

**Option C: Via ArgoCD Web UI**

1. Open ArgoCD UI: `http://argocd.k8s.home`
2. Click **+ New App**
3. Fill in:
   - Application Name: `nextcloud`
   - Project: `homelab`
   - Repository URL: `https://github.com/YOUR_USERNAME/YOUR_REPO.git`
   - Path: `nextcloud/overlays/longhorn`
   - Cluster: `https://kubernetes.default.svc`
   - Namespace: `nextcloud`
4. Click **Create**

### 3. Sync Application

```bash
# Force sync if needed
argocd app sync nextcloud

# Or via kubectl
kubectl patch application nextcloud -n argocd \
  --type merge \
  -p '{"operation": {"initiatedBy": {"username": "admin"}, "sync": {}}}'
```

---

## Verification

### 1. Check ArgoCD Status

```bash
# Application status
argocd app get nextcloud

# Or via kubectl
kubectl get application nextcloud -n argocd
```

**Expected:** `Status: Synced`, `Health: Healthy`

### 2. Check Pods

```bash
kubectl get pods -n nextcloud -w
```

**Expected output (after 2-5 minutes):**

```
NAME                         READY   STATUS    RESTARTS   AGE
nextcloud-xxxxxxxxxx-xxxxx   1/1     Running   0          2m
postgres-0                   1/1     Running   0          3m
redis-xxxxxxxxxx-xxxxx       1/1     Running   0          3m
```

### 3. Check Services

```bash
kubectl get svc -n nextcloud
```

### 4. Check PVCs

```bash
kubectl get pvc -n nextcloud
```

**Expected:** All PVCs should be `Bound`

### 5. Check HTTPRoute

```bash
kubectl get httproute -n nextcloud
```

### 6. Test Connectivity

```bash
# Test from cluster
kubectl run test --rm -it --image=curlimages/curl -- \
  curl -I http://nextcloud.nextcloud.svc.cluster.local

# Test from local machine
curl -I http://nextcloud.k8s.home
```

---

## Post-Installation

### 1. Access Nextcloud

Open in browser: **http://nextcloud.k8s.home**

Login with credentials from your secrets:

- Username: `admin` (or what you set)
- Password: (what you encrypted)

### 2. Initial Configuration

After first login:

1. **Skip app installation** (can add later)
2. Go to **Settings** → **Administration** → **Overview**
3. Review security warnings

### 3. Recommended Settings

```bash
# Enter Nextcloud pod
kubectl exec -it -n nextcloud deployment/nextcloud -- bash

# Run as www-data
su -s /bin/bash www-data

# Set background jobs to cron (already configured)
php occ background:cron

# Set default phone region
php occ config:system:set default_phone_region --value="US"

# Enable maintenance window
php occ config:system:set maintenance_window_start --type=integer --value=1
```

### 4. Install Recommended Apps

Via web UI or CLI:

```bash
kubectl exec -n nextcloud deployment/nextcloud -- \
  su -s /bin/bash www-data -c "php occ app:install calendar"

kubectl exec -n nextcloud deployment/nextcloud -- \
  su -s /bin/bash www-data -c "php occ app:install contacts"

kubectl exec -n nextcloud deployment/nextcloud -- \
  su -s /bin/bash www-data -c "php occ app:install tasks"
```

---

## Troubleshooting

### ArgoCD Sync Failed

```bash
# Check application events
kubectl describe application nextcloud -n argocd

# Check ArgoCD logs
kubectl logs -n argocd deployment/argocd-repo-server

# SOPS decryption issues
kubectl logs -n argocd deployment/argocd-repo-server | grep -i sops
```

**Common issues:**

- SOPS key not found → Verify `sops-age-key` secret exists in argocd namespace
- Repository not accessible → Check repo URL and credentials

### Pods Not Starting

```bash
# Describe pod for events
kubectl describe pod -n nextcloud <pod-name>

# Check logs
kubectl logs -n nextcloud <pod-name>

# Check PVC binding
kubectl get pvc -n nextcloud
```

**Common issues:**

- PVC pending → Longhorn might need more time or storage issue
- Init container waiting → Database not ready yet (normal, wait 1-2 min)

### Database Connection Failed

```bash
# Check PostgreSQL is running
kubectl get pods -n nextcloud -l app=postgres

# Check PostgreSQL logs
kubectl logs -n nextcloud postgres-0

# Test connection from Nextcloud pod
kubectl exec -n nextcloud deployment/nextcloud -- \
  pg_isready -h postgres -p 5432 -U nextcloud
```

### SOPS Decryption Issues

```bash
# Verify AGE secret exists
kubectl get secret sops-age-key -n argocd

# Check KSOPS is working
kubectl logs -n argocd deployment/argocd-repo-server | grep -i "age\|sops"

# Re-encrypt if needed
sops -d nextcloud/base/secrets.yaml > /tmp/secrets-plain.yaml
# Edit /tmp/secrets-plain.yaml
sops -e /tmp/secrets-plain.yaml > nextcloud/base/secrets.yaml
rm /tmp/secrets-plain.yaml
```

### Reset Deployment

```bash
# Delete via ArgoCD
argocd app delete nextcloud --cascade

# Or via kubectl
kubectl delete application nextcloud -n argocd
kubectl delete namespace nextcloud

# Wait for cleanup
kubectl get ns nextcloud  # Should return "not found"

# Re-deploy
kubectl apply -f ArgoCD/applications/nextcloud.yaml
```

---

## Quick Reference

### Key URLs

| Service   | URL                        |
| --------- | -------------------------- |
| Nextcloud | http://nextcloud.k8s.home  |
| ArgoCD    | http://argocd.k8s.home     |
| Longhorn  | http://longhorn.k8s.home   |
| Pi-hole   | http://192.168.68.10/admin |

### Key Commands

```bash
# Check status
kubectl get pods -n nextcloud
argocd app get nextcloud

# View logs
kubectl logs -n nextcloud deployment/nextcloud -f
kubectl logs -n nextcloud postgres-0 -f

# Nextcloud CLI
kubectl exec -n nextcloud deployment/nextcloud -- \
  su -s /bin/bash www-data -c "php occ <command>"

# Edit encrypted secrets
sops nextcloud/base/secrets.yaml

# Force ArgoCD sync
argocd app sync nextcloud --force
```

### IP Addresses

| Resource     | IP                 |
| ------------ | ------------------ |
| Master Node  | 192.168.68.120     |
| Worker Node  | 192.168.68.121     |
| Pi-hole DNS  | 192.168.68.10      |
| MetalLB Pool | 192.168.68.200-210 |
| Ingress LB   | 192.168.68.200     |

---

## Deployment Checklist

- [ ] Secrets edited with real passwords
- [ ] Secrets encrypted with SOPS
- [ ] Changes committed and pushed to GitHub
- [ ] DNS record added in Pi-hole
- [ ] ArgoCD application repo URL updated
- [ ] ArgoCD application applied
- [ ] All pods running
- [ ] Nextcloud accessible in browser
- [ ] Admin login successful
- [ ] Background jobs configured

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                         GitHub Repository                        │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  nextcloud/base/secrets.yaml (SOPS encrypted)           │    │
│  │  nextcloud/overlays/longhorn/kustomization.yaml         │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                      ArgoCD (GitOps)                             │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐       │
│  │  Repo Server │───▶│    KSOPS     │───▶│  Application │       │
│  │              │    │  (decrypt)   │    │  Controller  │       │
│  └──────────────┘    └──────────────┘    └──────────────┘       │
│         │                                        │               │
│         ▼                                        ▼               │
│  ┌──────────────┐                    ┌──────────────────┐       │
│  │ sops-age-key │                    │ Deploy to K8s    │       │
│  │   (Secret)   │                    │                  │       │
│  └──────────────┘                    └──────────────────┘       │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                            │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    nextcloud namespace                    │   │
│  │                                                           │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐       │   │
│  │  │  Nextcloud  │  │  PostgreSQL │  │    Redis    │       │   │
│  │  │ Deployment  │  │ StatefulSet │  │ Deployment  │       │   │
│  │  └──────┬──────┘  └──────┬──────┘  └─────────────┘       │   │
│  │         │                │                                │   │
│  │         ▼                ▼                                │   │
│  │  ┌─────────────┐  ┌─────────────┐                        │   │
│  │  │  Longhorn   │  │  Longhorn   │                        │   │
│  │  │  PVC 50Gi   │  │  PVC 10Gi   │                        │   │
│  │  └─────────────┘  └─────────────┘                        │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                │                                 │
│                                ▼                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              Envoy Gateway (HTTPRoute)                    │   │
│  │                  nextcloud.k8s.home                       │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                │                                 │
│                                ▼                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                MetalLB (192.168.68.200)                   │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Pi-hole DNS                                 │
│              nextcloud.k8s.home → 192.168.68.200                │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                        Browser                                   │
│                  http://nextcloud.k8s.home                       │
└─────────────────────────────────────────────────────────────────┘
```
