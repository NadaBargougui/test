# Aurora Console — dev-newhs Deployment Guide

> **Cluster:** `dev-newhs` (newhomeserver) — MicroCloud / MicroK8s  
> **Branch:** `deploy/dev-newhs`  
> **Workspace:** `aurora-console/test-managed-services/`  
> **Status:** Pre-production — isolated from org's `dev` and `prod` environments

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Prerequisites](#2-prerequisites)
3. [Phase 1 — Sealed Secrets](#3-phase-1--sealed-secrets)
4. [Phase 2 — Traefik Deployment](#4-phase-2--traefik-deployment)
5. [Phase 3 — Tailscale Operator](#5-phase-3--tailscale-operator)
6. [Phase 4 — Install ArgoCD](#6-phase-4--install-argocd)
7. [Phase 5 — Fix portal-api ConfigMap](#7-phase-5--fix-portal-api-configmap)
8. [Phase 6 — PostgreSQL](#8-phase-6--postgresql)
9. [Phase 7 — ArgoCD Bootstrap](#9-phase-7--argocd-bootstrap)
10. [Phase 8 — Fix Image Pull (ghcr-secret)](#10-phase-8--fix-image-pull-ghcr-secret)
11. [Isolation Strategy — Why test-managed-services](#11-isolation-strategy--why-test-managed-services)
12. [Pre-Bootstrap Checklist](#12-pre-bootstrap-checklist)

---

## 1. Architecture Overview

The `dev-newhs` environment is **never exposed to the public internet**. Access is controlled entirely through Tailscale.

```
Developer / Client machine (Tailscale installed)
        │
        │  private tailnet (100.x.x.x)
        ▼
Traefik LoadBalancer  ←  tailnet IP: 100.100.237.56
  hostname: traefik-newhs
        │
        ├── portal-newhs.auroraiq.cloud/api/ws  →  portal-api:8080  (WebSocket, HTTP/1.1)
        ├── portal-newhs.auroraiq.cloud/api      →  portal-api:8080  (REST API)
        ├── portal-newhs.auroraiq.cloud          →  portal-ui:443    (React frontend)
        └── auth-newhs.auroraiq.cloud            →  keycloak:8443    (Login / JWT)
```

### Key components and their roles

| Component | Role |
|---|---|
| **Tailscale Operator** | Assigns a private `100.x` IP to the Traefik LoadBalancer — replaces the need for a public load balancer |
| **Traefik** | Reverse proxy — receives HTTPS traffic and routes it to the correct pod based on hostname and path |
| **Keycloak** | Identity provider — owns the login page, issues JWT tokens, validates user identity |
| **portal-api** | Backend API — handles all resource provisioning requests, validates JWT tokens with Keycloak on every request |
| **portal-ui** | React frontend — the web portal users interact with |
| **PostgreSQL** | Database for portal-api — stores tenants, VMs, users, provisioning jobs |
| **ArgoCD** | GitOps engine — watches the git repo and keeps the cluster in sync with what's in git |
| **Sealed Secrets** | Encrypts Kubernetes Secrets so they can be safely stored in git |

> **Install order matters:** Traefik must be installed before the Tailscale Operator (the operator claims Traefik's LoadBalancer service to assign the tailnet IP). PostgreSQL must be running before ArgoCD bootstrap (portal-api depends on the database at startup).

---

## 2. Prerequisites

Before starting, make sure:

- [ ] You are on the correct server: `portal-01` (the node running the `dev-newhs` K8s cluster)
- [ ] `helm` is installed
- [ ] `kubectl` / `k8s kubectl` is available
- [ ] You have cloned `auroraiq-console-gitops` and are on the `deploy/dev-newhs` branch
- [ ] You have access to the Tailscale admin console at [login.tailscale.com](https://login.tailscale.com)
- [ ] `sealed-secrets-controller` is already running (installed separately)

```bash
# Verify sealed-secrets is running
kubectl get pods -n kube-system | grep sealed-secrets
```

**Expected output:**
```
sealed-secrets-controller-799bdfc45-75jkt   1/1   Running   0   2d8h
```

---

## 3. Phase 1 — Sealed Secrets

> **Why:** Kubernetes Secrets (passwords, tokens, credentials) cannot be stored in plain text in git. Sealed Secrets encrypts them using the cluster's own certificate — only that specific cluster can decrypt them. This means your git repo is safe to be public or shared.

```bash
# Apply all sealed secrets for the portal namespace
kubectl apply -f aurora-console/test-managed-services/sealed-secrets/portal/

# Apply ArgoCD repo credentials (needed for Phase 7)
kubectl apply -f aurora-console/test-managed-services/sealed-secrets/argocd/argocd-repo-credentials.yaml
```

> ⚠️ **Critical:** Sealed Secrets are encrypted against a **specific cluster's certificate**. If a SealedSecret was created on a different cluster (e.g. the old `dev` cluster), it will silently fail to decrypt here — `kubectl apply` will say "configured" but `kubectl get secret <name>` will return "not found". If this happens, you must re-seal the secret using this cluster's certificate (see [Phase 8](#10-phase-8--fix-image-pull-ghcr-secret) for the `ghcr-secret` case).

**Verify the DB secret was decrypted:**
```bash
kubectl get secret portal-db-credentials -n dev
```

**Expected output:**
```
NAME                    TYPE     DATA   AGE
portal-db-credentials   Opaque   2      7s
```

---

## 4. Phase 2 — Traefik Deployment

> **Why:** Traefik is the reverse proxy that sits in front of all services. It receives all incoming HTTPS traffic and routes it to the correct pod based on the hostname and URL path. It must be installed **before** the Tailscale Operator, because the operator's job is to assign a tailnet IP to Traefik's LoadBalancer service.

```bash
# Install Helm if not already installed
snap install helm --classic

# Add the Traefik chart repository
helm repo add traefik https://traefik.github.io/charts
helm repo update

# Install Traefik
helm -n traefik upgrade --install traefik traefik/traefik \
  --version 40.2.0 \
  -f aurora-console/dev-newhs/traefik/values.yaml \
  --create-namespace
```

**Verify Traefik is running:**
```bash
kubectl get svc -n traefik traefik
```

**Expected output (before Tailscale Operator):**
```
NAME      TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
traefik   LoadBalancer   10.152.183.195   <pending>     80:31384/TCP,443:30443/TCP   19s
```

> The `EXTERNAL-IP` showing `<pending>` is **expected at this stage**. The Tailscale Operator (Phase 3) will fill it in with the tailnet IP.

---

## 5. Phase 3 — Tailscale Operator

> **Why:** The Tailscale Operator runs inside the cluster and connects it to your Tailscale tailnet. Its main job here is to watch for LoadBalancer services annotated with `loadBalancerClass: tailscale` and assign them a private `100.x` tailnet IP. This is what makes Traefik reachable from any machine on the tailnet — no public internet exposure needed.

### Step 1 — Configure Tailscale ACL

Go to your [Tailscale Admin Console](https://login.tailscale.com) → **Access controls** → **JSON editor** and paste:

```json
{
  "grants": [
    {
      "src": ["*"],
      "dst": ["*"],
      "ip": ["*"]
    }
  ],
  "ssh": [
    {
      "action": "check",
      "src": ["autogroup:member"],
      "dst": ["autogroup:self"],
      "users": ["autogroup:nonroot", "root"]
    }
  ],
  "tagOwners": {
    "tag:k8s-operator": ["autogroup:admin", "tag:k8s-operator"],
    "tag:k8s-portal-dev": ["tag:k8s-operator"]
  }
}
```

> **Why the tags:** The `tag:k8s-operator` tag identifies the Tailscale Operator as a device on the tailnet. The `tagOwners` field defines who is allowed to assign that tag — admins and the operator itself. Without these tags defined in `tagOwners`, the operator will crash with a `400 — requested tags are invalid or not permitted` error.

### Step 2 — Create an OAuth Credential

Go to **Settings → Trust credentials → + Credential → OAuth**

Set the following scopes:

| Section | Permission | Tags |
|---|---|---|
| Devices → Core | **Write** | `tag:k8s-operator`, `tag:k8s-portal-dev` |
| Keys → Auth Keys | **Write** | `tag:k8s-operator`, `tag:k8s-portal-dev` |

Click **Generate credential** and save the `clientId` and `clientSecret`.

> **Why OAuth instead of an auth key:** An auth key is single-use and expires. The OAuth credential lets the operator generate its own auth keys dynamically and register new devices automatically as the cluster scales.

### Step 3 — Install the Tailscale Operator

```bash
helm repo add tailscale https://pkgs.tailscale.com/helmcharts
helm repo update

helm -n tailscale upgrade --install tailscale-operator tailscale/tailscale-operator \
  --version 1.98.4 \
  --create-namespace \
  -f /tmp/ts-values.yaml \
  --set oauth.clientId=<YOUR_CLIENT_ID> \
  --set oauth.clientSecret=<YOUR_CLIENT_SECRET>
```

**Watch operator logs to confirm startup:**
```bash
kubectl logs -n tailscale -l app=operator --follow
```

**Verify operator and Traefik proxy pods are running:**
```bash
kubectl get pods -n tailscale
```

**Expected output:**
```
NAME                          READY   STATUS    RESTARTS   AGE
operator-797d9c6dcc-prqjm     1/1     Running   0          107s
ts-traefik-x8hhw-0            1/1     Running   0          103s
```

**Verify Traefik now has a tailnet IP:**
```bash
kubectl get svc -n traefik traefik
```

**Expected output:**
```
NAME      TYPE           CLUSTER-IP       EXTERNAL-IP                                          PORT(S)                      AGE
traefik   LoadBalancer   10.152.183.195   100.100.237.56,traefik-newhs.tailefca39.ts.net      80:31384/TCP,443:30443/TCP   2d7h
```

**Verify on the tailnet:**
```bash
tailscale status
```

You should see both `tailscale-operator` and `traefik-newhs` as connected devices:
```
100.124.74.117   tailscale-operator   tagged-devices   linux   -
100.100.237.56   traefik-newhs        tagged-devices   linux   -
```

### Common Issue — CrashLoopBackOff

If the operator crashes with:
```
creating operator authkey: requested tags [tag:k8s-operator] are invalid or not permitted (400)
```

This is caused by one or more of these mistakes:

| Mistake | Fix |
|---|---|
| OAuth client has `devices:core:read` instead of `write` | Create a new OAuth client with `write` |
| `tagOwners` has `"tag:k8s-operator": []` (empty — no owner) | Add `"autogroup:admin"` to tagOwners |
| OAuth client was created before ACL was saved | Delete old clients, fix ACL first, create new client |

Fix: update the ACL, delete old OAuth clients, create a fresh one, then reinstall the operator:
```bash
helm -n tailscale uninstall tailscale-operator
helm -n tailscale upgrade --install tailscale-operator tailscale/tailscale-operator \
  --version 1.98.4 --create-namespace \
  -f /tmp/ts-values.yaml \
  --set oauth.clientId=<NEW_ID> \
  --set oauth.clientSecret=<NEW_SECRET>
```

---

## 6. Phase 4 — Install ArgoCD

> **Why:** ArgoCD is the GitOps engine. Once bootstrapped, it watches the git repository and automatically applies any changes you push — you never run `kubectl apply` manually again. It also enables disaster recovery: if the cluster is wiped, one `kubectl apply` of the bootstrap file restores everything.

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

helm install argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace \
  --set server.service.type=ClusterIP
```

**Verify all ArgoCD pods are running:**
```bash
kubectl get pods -n argocd
```

**Expected output:**
```
NAME                                                READY   STATUS    RESTARTS   AGE
argocd-application-controller-0                     1/1     Running   0          24h
argocd-applicationset-controller-5cfdf6dd79-65s86   1/1     Running   0          24h
argocd-dex-server-7b7cd94df4-sppbd                  1/1     Running   0          24h
argocd-notifications-controller-5d84ddcf-dtsqw      1/1     Running   0          24h
argocd-redis-85f5d89645-z58xf                       1/1     Running   0          24h
argocd-repo-server-8564479d64-5rkrz                 1/1     Running   0          24h
argocd-server-79fd69df95-ksr4s                      1/1     Running   0          24h
```

---

## 7. Phase 5 — Fix portal-api ConfigMap

> **Why:** The `portal-api` talks to MicroCloud (LXD) to provision VMs. Its ConfigMap must contain values that match what actually exists on the real MicroCloud — otherwise VM provisioning will silently fail.

**Check current values:**
```bash
cat aurora-console/test-managed-services/portal-api/configmap.yaml
```

**Verify against the real MicroCloud on server4:**
```bash
# SSH into server4, then:
lxc storage list          # confirms storage pool name
lxc network list          # confirms UPLINK exists
lxc project list          # confirms auroraiq-portal project exists
lxc network list --project auroraiq-portal  # confirms portal-net exists
```

**Fix the storage pool name mismatch** (was `disks`, should be `local`):
```bash
sed -i 's/LXD_STORAGE_POOL: "disks"/LXD_STORAGE_POOL: "local"/' \
  aurora-console/test-managed-services/portal-api/configmap.yaml

# Verify the fix
grep LXD_STORAGE_POOL aurora-console/test-managed-services/portal-api/configmap.yaml
```

**Expected output:**
```
LXD_STORAGE_POOL: "local"
```

**Push the fix:**
```bash
git add aurora-console/test-managed-services/portal-api/configmap.yaml
git commit -m "test-managed-services: fix LXD_STORAGE_POOL from disks to local"
git push origin deploy/dev-newhs
```

---

## 8. Phase 6 — PostgreSQL

> **Why:** `portal-api` connects to PostgreSQL at startup. If the database is not running when ArgoCD deploys the app, portal-api will crash in a boot loop. PostgreSQL must be up **before** the ArgoCD bootstrap.
>
> PostgreSQL is installed via Helm manually (not managed by ArgoCD) because it requires persistent state that should not be pruned or replaced by GitOps automation.

**Apply the database credentials SealedSecret first:**
```bash
kubectl apply -f aurora-console/test-managed-services/sealed-secrets/portal/portal-db-credentials.yaml

# Wait 10 seconds for decryption, then verify
sleep 10
kubectl get secret portal-db-credentials -n dev
```

**Expected output:**
```
NAME                    TYPE     DATA   AGE
portal-db-credentials   Opaque   2      7s
```

**Create the namespace and install PostgreSQL:**
```bash
kubectl apply -f aurora-console/test-managed-services/namespace.yaml

helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

helm install portal-db bitnami/postgresql -n dev \
  -f aurora-console/test-managed-services/postgresql/values.yaml
```

**Verify PostgreSQL is running:**
```bash
kubectl get pods -n dev | grep postgresql
```

**Expected output:**
```
portal-db-postgresql-0   1/1   Running   0   66s
```

> If the pod is stuck in `ContainerCreating` and you see `MountVolume.SetUp failed: secret "portal-db-credentials" not found`, the SealedSecret was not decrypted. Re-apply it and wait before proceeding.

---

## 9. Phase 7 — ArgoCD Bootstrap

> **Why:** This is the step that puts the cluster under full GitOps control. You apply one file manually — the root Application — and ArgoCD takes it from there: it finds the AppProject and child Application in git, creates them, then syncs all the portal manifests (portal-api, portal-ui, keycloak, ingress, RBAC, sealed secrets) automatically.

### Pre-Bootstrap Checklist

Run these checks before proceeding. All must pass:

```bash
# 1. sealed-secrets running?
kubectl get pods -n kube-system | grep sealed-secrets

# 2. Traefik running with tailnet IP?
kubectl get svc -n traefik traefik

# 3. Tailscale operator running?
kubectl get pods -n tailscale

# 4. ArgoCD running?
kubectl get pods -n argocd | grep argocd-server

# 5. PostgreSQL running?
kubectl get pods -n dev | grep postgresql
```

All five must show `Running`. Do not proceed if any are missing or in error state.

### Bootstrap

```bash
# Apply the ArgoCD repo credentials secret (so ArgoCD can clone the private repo)
kubectl apply -f aurora-console/test-managed-services/sealed-secrets/argocd/argocd-repo-credentials.yaml

# Apply the root Application (the only manual kubectl apply you will ever do for this env)
kubectl apply -f argocd/bootstrap/test-managed-services.yaml
```

**Check ArgoCD applications:**
```bash
kubectl get applications -n argocd
```

**Expected output:**
```
NAME                                   SYNC STATUS   HEALTH STATUS
root-app-test-managed-services         Synced        Healthy
aurora-console-test-managed-services   Synced        Degraded
```

> The root app being `Synced + Healthy` means ArgoCD successfully read the git repo and created the AppProject and child Application. The child app showing `Degraded` is normal at this point — it means ArgoCD synced the manifests but some pods are not healthy yet (image pull issues, etc.).

### Common Issue — `SYNC STATUS: Unknown` / Authentication required

If the root app shows `Unknown` sync status:

```bash
kubectl describe application root-app-test-managed-services -n argocd | tail -20
```

If you see `authentication required: Repository not found`, the repo credentials secret exists but is missing the label ArgoCD needs to recognize it:

```bash
# Check if the label is present
kubectl get secret argocd-repo-credentials -n argocd -o jsonpath='{.metadata.labels}'

# If empty output — add the label
kubectl label secret argocd-repo-credentials -n argocd \
  argocd.argoproj.io/secret-type=repository

# Force ArgoCD to re-read credentials
kubectl rollout restart deployment/argocd-repo-server -n argocd

# Wait 30s then check again
kubectl get applications -n argocd
```

---

## 10. Phase 8 — Fix Image Pull (ghcr-secret)

> **Why:** `portal-api`, `portal-ui`, and `keycloak` all pull their images from `ghcr.io/auroraiq-labs/` which is a **private** GitHub Container Registry. Kubernetes needs a `docker-registry` type secret called `ghcr-secret` in the `dev` namespace to authenticate. Without it, all three pods stay in `ImagePullBackOff` and the portal never comes up.

### Check current pod status

```bash
kubectl get pods -n dev
```

If you see:
```
keycloak-xxx      0/1   ImagePullBackOff       0   7m
portal-api-xxx    0/1   Init:ImagePullBackOff  0   7m
portal-ui-xxx     0/1   ImagePullBackOff       0   7m
portal-db-xxx     1/1   Running                0   20m
```

This confirms the image pull problem.

### Why the SealedSecret may not decrypt

The `ghcr-secret` is stored in git as a SealedSecret. If it was **sealed against a different cluster's certificate** (e.g. the old `dev` cluster), the sealed-secrets-controller on this cluster cannot decrypt it — and the secret is silently never created.

```bash
# Check if the secret exists
kubectl get secret ghcr-secret -n dev
```

If "not found" → the SealedSecret is not decrypting. You need to re-seal it against this cluster's certificate.

### Fix — Re-seal ghcr-secret against this cluster

```bash
# Get this cluster's sealed-secrets public certificate
kubeseal --fetch-cert \
  --controller-name=sealed-secrets-controller \
  --controller-namespace=kube-system \
  > /tmp/pub-cert.pem

# Create the raw secret (replace values with real credentials)
kubectl create secret docker-registry ghcr-secret \
  --docker-server=ghcr.io \
  --docker-username=<GITHUB_USERNAME> \
  --docker-password=<GITHUB_PAT> \
  --docker-email=<EMAIL> \
  --namespace=dev \
  --dry-run=client -o yaml > /tmp/ghcr-secret-raw.yaml

# Seal it with this cluster's cert
kubeseal --format=yaml \
  --cert=/tmp/pub-cert.pem \
  < /tmp/ghcr-secret-raw.yaml \
  > aurora-console/test-managed-services/sealed-secrets/portal/ghcr-secret.yaml

# Apply it
kubectl apply -f aurora-console/test-managed-services/sealed-secrets/portal/ghcr-secret.yaml

# Verify decryption (wait 10s)
sleep 10
kubectl get secret ghcr-secret -n dev
```

### Fix — Authorize the GitHub PAT for the org (403 Forbidden)

If the secret decrypts correctly but pods still fail with `403 Forbidden` from `ghcr.io`:

The GitHub Personal Access Token (PAT) is not SSO-authorized for the `auroraiq-labs` organization. Having `read:packages` scope is not enough — PATs must be explicitly authorized per organization when SSO is enabled.

1. Go to [github.com/settings/tokens](https://github.com/settings/tokens)
2. Find the PAT used in `ghcr-secret`
3. Click **"Configure SSO"** next to it
4. Click **"Authorize"** next to `auroraiq-labs`
5. Re-create and re-seal the secret with the same PAT (the token itself hasn't changed, but GitHub now allows it)
6. Re-apply the SealedSecret

**Verify pods are now pulling images:**
```bash
kubectl get pods -n dev --watch
```

**Expected final state:**
```
portal-db-postgresql-0   1/1   Running   0   20m
keycloak-xxx             1/1   Running   0   2m
portal-api-xxx           1/1   Running   0   2m
portal-ui-xxx            1/1   Running   0   2m
```

---

## 11. Isolation Strategy — Why test-managed-services

This section explains why work happens in `aurora-console/test-managed-services/` and not directly in `aurora-console/dev-newhs/`.

### The problem with modifying dev-newhs directly

`aurora-console/dev-newhs/` and `argocd/bootstrap/dev-newhs.yaml` already exist on `main` and are actively used by the organization's existing `dev-newhs` environment. Modifying those files on `deploy/dev-newhs` branch creates a **merge conflict** when merging back to `main` — git sees two versions of the same file and cannot automatically resolve it. Resolving it manually risks breaking the org's live environment.

### The solution

Create completely new files that nobody else is using:

```
aurora-console/test-managed-services/          ← your working copy of the manifests
argocd/bootstrap/test-managed-services.yaml    ← your ArgoCD root app
argocd/projects/aurora-console-test-managed-services.yaml
argocd/apps/test-managed-services/aurora-console.yaml
```

The original org files are **restored to their main state** on the branch — so merging `deploy/dev-newhs` → `main` later produces **zero conflicts**.

### Branch and folder ownership

| File/Folder | Branch | Owner |
|---|---|---|
| `aurora-console/test-managed-services/` | `deploy/dev-newhs` | You — isolated workspace |
| `argocd/bootstrap/test-managed-services.yaml` | `deploy/dev-newhs` | You — ArgoCD bootstrap |
| `argocd/projects/aurora-console-test-managed-services.yaml` | `deploy/dev-newhs` | You — AppProject |
| `argocd/apps/test-managed-services/aurora-console.yaml` | `deploy/dev-newhs` | You — Application |
| `aurora-console/dev-newhs/` | `main` | Org — untouched |
| `argocd/bootstrap/dev-newhs.yaml` | `main` | Org — untouched |

### How the ArgoCD App of Apps works

```
You (once, manually):
  kubectl apply -f argocd/bootstrap/test-managed-services.yaml
        │
        │  root app scans argocd/ folder (excluding bootstrap/, apps/dev/, apps/prod/, apps/dev-newhs/)
        ▼
ArgoCD creates:
  AppProject: aurora-console-test-managed-services   (from argocd/projects/)
  Application: aurora-console-test-managed-services  (from argocd/apps/test-managed-services/)
        │
        │  child app scans aurora-console/test-managed-services/
        ▼
ArgoCD deploys:
  portal-api, portal-ui, keycloak, ingress, RBAC, sealed secrets...
```

After this, **every `git push` to `deploy/dev-newhs` is a deployment**. No more `kubectl apply` needed.

---

## 12. Pre-Bootstrap Checklist

Quick reference table before running ArgoCD bootstrap:

| Check | Command | Expected |
|---|---|---|
| sealed-secrets | `kubectl get pods -n kube-system \| grep sealed-secrets` | `1/1 Running` |
| Traefik + tailnet IP | `kubectl get svc -n traefik traefik` | `EXTERNAL-IP = 100.100.237.56,...` |
| Tailscale operator | `kubectl get pods -n tailscale` | `operator` and `ts-traefik` both `Running` |
| ArgoCD | `kubectl get pods -n argocd \| grep argocd-server` | `1/1 Running` |
| PostgreSQL | `kubectl get pods -n dev \| grep postgresql` | `1/1 Running` |
| ghcr-secret | `kubectl get secret ghcr-secret -n dev` | `Opaque` type, not "not found" |

All six must be green before bootstrapping.

---

*Last updated: June 2026 — dev-newhs deployment on newhomeserver (MicroCloud / MicroK8s)*
