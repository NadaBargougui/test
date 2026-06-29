# Aurora Console — dev-newhs Deployment Guide

This guide documents every step required to bring up the dev-newhs environment on a new cluster, from infrastructure prerequisites (Traefik, Tailscale) through GitOps bootstrap (ArgoCD), and must be completed before moving on to aurora-console/dev-newhs/DEPLOY.md, which assumes all of these components are already running.

Each phase includes the reasoning behind it, the exact commands to run, and the expected output so you can verify success before proceeding to the next step.

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Phase 1 — Sealed Secrets](#2-phase-1--sealed-secrets)
3. [Phase 2 — Traefik Deployment](#3-phase-2--traefik-deployment)
4. [Phase 3 — Tailscale Operator](#4-phase-3--tailscale-operator)
5. [Phase 4 — Install ArgoCD](#5-phase-4--install-argocd)
6. [Isolation Strategy — Why test-managed-services](#6-isolation-strategy--why-test-managed-services)
7. [Phase 5 — Fix portal-api ConfigMap](#7-phase-5--fix-portal-api-configmap)
8. [Phase 6 — PostgreSQL](#8-phase-6--postgresql)
9. [Phase 7 — ArgoCD Bootstrap](#9-phase-7--argocd-bootstrap)

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
        ├── portal-newhs.auroraiq.cloud/api/ws   →  portal-api:8080  (WebSocket, HTTP/1.1)
        ├── portal-newhs.auroraiq.cloud/api      →  portal-api:8080  (REST API)
        ├── portal-newhs.auroraiq.cloud          →  portal-ui:443    (React frontend)
        └── auth-newhs.auroraiq.cloud            →  keycloak:8443    (Login / JWT)
```
> 100.100.237.56 is an example!

### Key components and their roles

| Component | Role |
|---|---|
| **Tailscale Operator** | Assigns a private `100.x` IP to the Traefik LoadBalancer — replaces the need for a public load balancer |
| **Traefik** | Reverse proxy — receives HTTPS traffic and routes it to the correct pod based on hostname and path |
| **Keycloak** | Identity provider — owns the login page, issues JWT tokens, validates user identity |
| **portal-api** | Backend API — handles all resource provisioning requests, validates JWT tokens with Keycloak on every request |
| **portal-ui** | React frontend — the web portal users interact with |
| **PostgreSQL** | Database for portal-api |
| **ArgoCD** | GitOps engine — watches the git repo and keeps the cluster in sync with what's in git |
| **Sealed Secrets** | Encrypts Kubernetes Secrets so they can be safely stored in git |

> **Install order matters:** Traefik must be installed before the Tailscale Operator (the operator claims Traefik's LoadBalancer service to assign the tailnet IP). PostgreSQL must be running before ArgoCD bootstrap (portal-api depends on the database at startup).

---

## 2. Phase 1 — Sealed Secrets

> **Why:** Kubernetes Secrets (passwords, tokens, credentials) cannot be stored in plain text in git. Sealed Secrets encrypts them using the cluster's own certificate, only that specific cluster can decrypt them. This means your git repo is safe to be public or shared.

Script coming soon..

---

## 3. Phase 2 — Traefik Deployment

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

## 4. Phase 3 — Tailscale Operator

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
NAME      TYPE           CLUSTER-IP       EXTERNAL-IP                                        PORT(S)                      AGE
traefik   LoadBalancer   10.152.183.195   100.100.237.56,traefik-newhs.tailefca39.ts.net    80:31384/TCP,443:30443/TCP   2d7h
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

---

## 5. Phase 4 — Install ArgoCD

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

## 6. Isolation Strategy — Why test-managed-services

> **Read this before touching any files.** This explains the folder and branch structure you will be working in for all remaining phases. It explains why we create a separate deployment folder instead of modifying existing `dev-newhs` resources.


### The problem
`portal-api/deployment.yaml` currently contains the old IP: 100.99.0.11
This needs to be updated to the new IP, and `portal-api/configmap.yaml` also requires changes (see Phase 5).

However, these files are part of the existing `aurora-console/dev-newhs` environment and are already used by other workloads. Modifying them directly could affect existing ArgoCD deployments.

A possible approach would be to create a `deploy/dev-newhs` branch, modify the existing `aurora-console/dev-newhs` files, and point ArgoCD to that branch.

However, this would create a **merge conflict** when merging back to `main` later.

### The solution — a completely isolated environment

Instead of modifying existing resources, we create a new environment with dedicated files:

```
aurora-console/test-managed-services/                         ← isolated copy of manifests
argocd/bootstrap/test-managed-services.yaml                   ← your ArgoCD root app
argocd/projects/aurora-console-test-managed-services.yaml     ← your AppProject
argocd/apps/test-managed-services/aurora-console.yaml         ← your Application
```

### Branch and folder ownership

| File/Folder | Branch | Owner |
|---|---|---|
| `aurora-console/test-managed-services/` | `deploy/dev-newhs` | You — isolated workspace |
| `argocd/bootstrap/test-managed-services.yaml` | `deploy/dev-newhs` | You — ArgoCD bootstrap |
| `argocd/projects/aurora-console-test-managed-services.yaml` | `deploy/dev-newhs` | You — AppProject |
| `argocd/apps/test-managed-services/aurora-console.yaml` | `deploy/dev-newhs` | You — Application |
| `aurora-console/dev-newhs/` | `main` | Org — untouched |
| `argocd/bootstrap/dev-newhs.yaml` | `main` | Org — untouched |

### How it was set up

```bash
# 1. Create the feature branch
git checkout -b deploy/dev-newhs

# 2. Copy dev-newhs into the new isolated folder
cp -r aurora-console/dev-newhs aurora-console/test-managed-services

# 3. Create the ArgoCD bootstrap, project, and app files from the dev-newhs originals
cp argocd/bootstrap/dev-newhs.yaml argocd/bootstrap/test-managed-services.yaml
cp argocd/projects/aurora-console-dev-newhs.yaml argocd/projects/aurora-console-test-managed-services.yaml
mkdir -p argocd/apps/test-managed-services
cp argocd/apps/dev-newhs/aurora-console.yaml argocd/apps/test-managed-services/aurora-console.yaml

# 4. Replace all dev-newhs references with test-managed-services in the new files
sed -i 's/dev-newhs/test-managed-services/g' argocd/bootstrap/test-managed-services.yaml
sed -i 's/dev-newhs/test-managed-services/g' argocd/projects/aurora-console-test-managed-services.yaml
sed -i 's/dev-newhs/test-managed-services/g' argocd/apps/test-managed-services/aurora-console.yaml

# 5. Update IP in test-managed-services/portal-api/deployment.yaml
sed -i 's|100.99.0.11|100.100.237.56|' aurora-console/test-managed-services/portal-api/deployment.yaml

# 6. Point the app and bootstrap to the deploy/dev-newhs branch (not main)
sed -i 's/targetRevision: main/targetRevision: deploy\/dev-newhs/' \
  argocd/apps/test-managed-services/aurora-console.yaml

sed -i 's/targetRevision: main/targetRevision: deploy\/dev-newhs/' \
  argocd/bootstrap/test-managed-services.yaml

# 7. Exclude the org's dev-newhs apps from your ArgoCD so it doesn't try to apply them
sed -i "s/exclude: '{bootstrap\/*,apps\/dev\/*,apps\/prod\/*}'/exclude: '{bootstrap\/*,apps\/dev\/*,apps\/prod\/*,apps\/dev-newhs\/*}'/" \
  argocd/bootstrap/test-managed-services.yaml

# 8. Push
git push origin deploy/dev-newhs
```

### How the ArgoCD App of Apps works

```
You (once, manually):
  kubectl apply -f argocd/bootstrap/test-managed-services.yaml
        │
        │  root app scans argocd/ folder
        │  (excludes bootstrap/, apps/dev/, apps/prod/, apps/dev-newhs/)
        ▼
ArgoCD creates:
  AppProject  → aurora-console-test-managed-services  (from argocd/projects/)
  Application → aurora-console-test-managed-services  (from argocd/apps/test-managed-services/)
        │
        │  child app scans aurora-console/test-managed-services/
        ▼
ArgoCD deploys:
  portal-api, portal-ui, keycloak, ingress...
```

After this, **every `git push` to `deploy/dev-newhs` is a deployment**. No more `kubectl apply` needed.

**Note:** After merging into `main`, you can safely change `targetRevision` back to `main`.

---

## 7. Phase 5 — Fix portal-api ConfigMap

> **Why:** The `portal-api` talks to MicroCloud (LXD) to provision VMs. Its ConfigMap must contain values that match what actually exists on the real MicroCloud — otherwise VM provisioning will silently fail. All changes from here on happen inside `aurora-console/test-managed-services/` (see [Isolation Strategy](#6-isolation-strategy--why-test-managed-services) above).

**Check current values:**
```bash
cat aurora-console/test-managed-services/portal-api/configmap.yaml
```

**Verify against the real MicroCloud on server4:**
```bash
# SSH into server4, then run:
lxc storage list                              # confirms storage pool name
lxc network list                              # confirms UPLINK exists
lxc project list                              # confirms auroraiq-portal exists
lxc network list --project auroraiq-portal    # confirms portal-net exists
```

> The only mismatch found was `LXD_STORAGE_POOL: "disks"` — the real pool is named `local`.

**Fix the storage pool name:**
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
> PostgreSQL is installed via Helm manually (not managed by ArgoCD) because it holds persistent state that should never be pruned or replaced by GitOps automation.

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

---
 
## 9. Phase 7 — ArgoCD Bootstrap (To be continued..)

> **Why:** This is the step that puts the cluster under full GitOps control. You apply one file manually — the root Application — and ArgoCD takes it from there: it finds the AppProject and child Application in git, creates them, then syncs all the portal manifests (portal-api, portal-ui, keycloak, ingress..) automatically.

### Pre-Bootstrap Checklist

Run all five checks before proceeding:

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
# Apply the root Application — the only manual kubectl apply for this environment
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
aurora-console-test-managed-services   Synced        Healthy
```

> The root app being `Synced + Healthy` means ArgoCD successfully read the git repo and created the AppProject and child Application. 

