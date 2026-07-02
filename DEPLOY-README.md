# Aurora Console — dev-newhs Deployment Guide

This guide documents every step required to bring up the dev-newhs environment on a new cluster, from infrastructure prerequisites (Traefik, Tailscale) through GitOps bootstrap (ArgoCD).

Each phase includes the reasoning behind it, the exact commands to run, and the expected output so you can verify success before proceeding to the next step.

> ## Before you start — values specific to this environment
>
> This guide was written while setting up **one particular** environment. Several names and one IP address below are **not generic placeholders** — they're the actual values chosen for that setup. If you're following these steps for your own environment, **replace them with your own names**, and keep them consistent everywhere they appear (branch, folders, and file names must all match each other).
>
> | Value used in this guide | What it is | Replace with |
> |---|---|---|
> | `deploy/dev-newhs` | Git branch name | Your own branch name |
> | `aurora-console/test-managed-services/` | Isolated manifests folder | Your own environment folder name |
> | `argocd/bootstrap/test-managed-services.yaml` | ArgoCD root application file | Match your environment folder name |
> | `argocd/projects/aurora-console-test-managed-services.yaml` | ArgoCD AppProject file | Match your environment folder name |
> | `argocd/apps/test-managed-services/aurora-console.yaml` | ArgoCD Application file | Match your environment folder name |
> | `100.100.237.56` | Tailscale IP assigned to Traefik | **This will be different for every setup.** Get it from `kubectl get svc -n traefik traefik` after Phase 3. |
> | `admin` (Phase 8) | Initial portal administrator username | Your own choice. Change it if you want a different username. |
> | `admin@auroraiq.cloud` (Phase 8) | Email address for the initial portal administrator | Your own email address |
> | `admin@auroraiq.cloud` (Phase 9, Brevo SMTP) | Verified SMTP **From** sender address | A real mailbox you control on your domain. This is **independent of the Phase 8 admin user**. |
>
> Everywhere `test-managed-services` appears (folder names, file names, ArgoCD app/project names), it must be the **same string** you chose — that's what ties the branch, the folders, and the ArgoCD objects together. Mixing different names between steps will break the bootstrap.

---

## Table of Contents

0. [Isolation Strategy — Why test-managed-services](#0-isolation-strategy--why-test-managed-services)
1. [Architecture Overview](#1-architecture-overview)
2. [Phase 1 — Sealed Secrets](#2-phase-1--sealed-secrets)
3. [Phase 2 — Traefik Deployment](#3-phase-2--traefik-deployment)
4. [Phase 3 — Tailscale Operator](#4-phase-3--tailscale-operator)
5. [Phase 4 — Install ArgoCD](#5-phase-4--install-argocd)
6. [Phase 5 — Fix portal-api ConfigMap](#6-phase-5--fix-portal-api-configmap)
7. [Phase 6 — PostgreSQL](#7-phase-6--postgresql)
8. [Phase 7 — ArgoCD Bootstrap](#8-phase-7--argocd-bootstrap)
9. [Phase 8 — Create the first portal admin user](#9-phase-8--create-the-first-portal-admin-user)
10. [Phase 9 — Configure Keycloak SMTP (Brevo) from scratch](#10-phase-9--configure-keycloak-smtp-brevo-from-scratch)

---

## 0. Isolation Strategy — Why test-managed-services

> **Read this before touching any files.** This explains the folder and branch structure you will be working in for all remaining phases, and why we create a separate deployment folder instead of modifying existing `dev-newhs` resources.

### The problem

We're going to need to change some files in the repo in upcoming steps. However, these files are already used by other workloads. Modifying them directly could affect existing ArgoCD deployments.

A possible approach would be to create a `deploy/dev-newhs` branch, modify the existing files, and point ArgoCD to that branch. However, this would create a **merge conflict** when merging back to `main` later.

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
# 0. Clone the repo
git clone https://github.com/AuroraIQ-labs/auroraiq-console-gitops.git
cd auroraiq-console-gitops

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

**Note:** After merging into `main`, you can safely change `targetRevision` back to `main`.

---

## 1. Architecture Overview

The `dev-newhs` environment is **never exposed to the public internet**. Access is controlled entirely through Tailscale.

```
Developer / Client machine (Tailscale installed)
        │
        │  private tailnet (100.x.x.x)
        ▼
Traefik LoadBalancer  ←  tailnet IP: 100.x.x.x
  hostname: traefik-newhs
        │
        ├── portal-newhs.auroraiq.cloud/api/ws   →  portal-api:8080  (WebSocket, HTTP/1.1)
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
| **PostgreSQL** | Database for portal-api |
| **ArgoCD** | GitOps engine — watches the git repo and keeps the cluster in sync with what's in git |
| **Sealed Secrets** | Encrypts Kubernetes Secrets so they can be safely stored in git |

> **Install order matters:** Traefik must be installed before the Tailscale Operator (the operator claims Traefik's LoadBalancer service to assign the tailnet IP). PostgreSQL must be running before ArgoCD bootstrap (portal-api depends on the database at startup).

---

## 2. Phase 1 — Sealed Secrets

> **Why:** Kubernetes Secrets (passwords, tokens, credentials) cannot be stored in plain text in git. Sealed Secrets encrypts them using the cluster's own certificate — only that specific cluster can decrypt them. This means your git repo is safe to be public or shared.

```bash
# 1. Install the sealed-secrets controller, then export its public cert:
kubeseal --controller-namespace=kube-system \
         --controller-name=sealed-secrets-controller \
         --fetch-cert > aurora-console/test-managed-services/sealed-secrets/dev-newhs-pub-cert.pem

git push -u origin deploy/dev-newhs
```

> When deploying the sealed-secrets controller in the cluster, it generates one **public cert** used to encrypt secrets and one **private key** used to decrypt them before storing them in etcd. We need to fetch the public cert and put it in our repo.

*Script for creating all the secrets is coming soon.*

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
  -f aurora-console/test-managed-services/traefik/values.yaml \
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
> 100.100.237.56 is an example — yours will be a different tailnet IP, assigned automatically by the Tailscale Operator. Use whatever IP shows up in your own output.

**Verify on the tailnet:**
```bash
tailscale status
```

You should see both `tailscale-operator` and `traefik-newhs` as connected devices:
```
100.124.74.117   tailscale-operator   tagged-devices   linux   -
100.100.237.56   traefik-newhs        tagged-devices   linux   -
```

### Step 4 — Update IP in test-managed-services/portal-api/deployment.yaml

```bash
sed -i 's|100.99.0.11|100.100.237.56|' aurora-console/test-managed-services/portal-api/deployment.yaml

git push origin deploy/dev-newhs
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

## 6. Phase 5 — Fix portal-api ConfigMap

> **Why:** The `portal-api` talks to MicroCloud (LXD) to provision VMs. Its ConfigMap must contain values that match what actually exists on the real MicroCloud — otherwise VM provisioning will silently fail. 

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

## 7. Phase 6 — PostgreSQL

> **Why:** `portal-api` connects to PostgreSQL at startup. If the database is not running when ArgoCD deploys the app, portal-api will crash in a boot loop. PostgreSQL must be up **before** the ArgoCD bootstrap.
>
> PostgreSQL is installed via Helm manually (not managed by ArgoCD) because it holds persistent state that should never be pruned or replaced by GitOps automation.

**Verify secret is there first:**
```bash
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

## 8. Phase 7 — ArgoCD Bootstrap

> **Why:** This is the step that puts the cluster under full GitOps control. You apply one file manually — the root Application — and ArgoCD takes it from there: it finds the AppProject and child Application in git, creates them, then syncs all the portal manifests (portal-api, portal-ui, keycloak, ingress...) automatically.

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

### How the ArgoCD works

```
You (once, manually):
  kubectl apply -f argocd/bootstrap/test-managed-services.yaml
        │
        │  root app scans argocd/ folder
        │  (excludes bootstrap/, apps/dev/, apps/prod/, apps/test-managed-services/)
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

### Bootstrap

```bash
# Apply the root Application — the only manual kubectl apply for this environment
kubectl apply -f aurora-console/test-managed-services/sealed-secrets/argocd/argocd-repo-credentials.yaml
kubectl apply -f argocd/bootstrap/test-managed-services.yaml
```

> Note: you need access to the images in packages (GitHub Container Registry) in order to pull the Docker images!

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

**Check ArgoCD if ArgoCD deployed our apps:**
```bash
kubectl get pods -n dev
```

**Expected output:**
```
NAME                          READY   STATUS    
keycloak-848496fdbb-mzm7w     1/1     Running   
portal-api-747fd686d4-tk6np   1/1     Running   
portal-db-postgresql-0        1/1     Running 
portal-ui-77bb7f46c6-94sm5    1/1     Running   
portal-api-747fd686d4-tk6np   1/1     Running  
keycloak-848496fdbb-mzm7w     1/1     Running  
keycloak-848496fdbb-mzm7w     1/1     Running   
portal-api-747fd686d4-tk6np   1/1     Running   
portal-api-747fd686d4-tk6np   1/1     Running  
keycloak-848496fdbb-mzm7w     1/1     Running   
keycloak-848496fdbb-mzm7w     1/1     Running   
keycloak-848496fdbb-mzm7w     1/1     Running   
```

---

## 9. Phase 8 — Create the first portal admin user

The portal's admin pages (the *backoffice*) are gated on the Keycloak **`admin` realm role** (`kaas-api` `adminOnly` middleware checks `realm_access.roles`). The `auroraiq` realm ships with **no seeded users** (`registrationAllowed: true`), so you must create the first admin manually and assign it that role. Signup-approval is a **separate** flow and does NOT gate admin access.

> ⚠️ Without this phase, NO ONE can access the backoffice. Even logging in as `admin` from the Keycloak master console gives you nothing on the portal — `master/admin` is the Keycloak super-user, NOT a portal user. The portal only reads tokens from the `auroraiq` realm.

> **Note:** the steps below use `admin` as the example username for the first portal user (not to be confused with the Keycloak master-realm super-user mentioned above — this is a separate user created inside the `auroraiq` realm). Change it to whatever username you'd like — just keep it consistent between the "create user", "set password", and "assign role" steps.

### 9.1 — Option A: Keycloak admin console (UI, step by step)

1. **Open the Keycloak admin console** → https://auth-newhs.auroraiq.cloud/admin
   - Login as `admin` / `<KC_BOOTSTRAP_ADMIN_PASSWORD>` (from `keycloak-secrets`).

2. **Switch to the right realm** — top-left realm dropdown → select **`auroraiq`** (NOT `master` — `master` is only for Keycloak admin itself).

3. **Left sidebar → `Users` → button `Add user` (top-right).**

4. **Fill the Create user form:**
   - **Username**: `admin` (or your choice — this is what you'll type at login)
   - **Email**: `admin@auroraiq.cloud`
   - **Email verified**: toggle **On** (skip the verify-mail step)
   - **First name / Last name**: optional, fill if you want it shown in the portal
   - Leave **Required user actions** empty (otherwise the user is forced to UPDATE_PASSWORD / VERIFY_EMAIL on first login)
   - Click **Create** → you land on the user detail page.

5. **Tab `Credentials` → button `Set password`:**
   - **Password**: `<your password>`
   - **Password confirmation**: same
   - **Temporary**: toggle **Off** (otherwise the user must change it on first login)
   - Click **Save** → confirm in the modal.

6. **Tab `Role mapping` → button `Assign role`:**
   - The dialog opens with a **Filter** dropdown at the top-left.
   - Click the dropdown → select **`Filter by realm roles`** (NOT client roles — the `kaas-api` middleware checks the *realm* role).
   - In the search box, type `admin` → the realm role `admin` appears.
   - **Check the line `admin`** → click **Assign**.
   - The role now shows in the user's Role mapping list.

7. **Log out / log back in on the portal** (https://portal-newhs.auroraiq.cloud) so the new token carries the role. The **Admin** menu appears in the left sidebar.

### 9.2 — Option B: `kcadm` CLI (scriptable, idempotent for repeat envs)

```bash
KC=deploy/keycloak
ADMIN_PW=$(kubectl get secret keycloak-secrets -n dev -o jsonpath='{.data.KC_BOOTSTRAP_ADMIN_PASSWORD}' | base64 -d)

# Authenticate kcadm against the master realm
kubectl exec -n dev $KC -- /opt/keycloak/bin/kcadm.sh config credentials \
  --server http://localhost:8080 --realm master --user admin --password "$ADMIN_PW"

# Create the user in the auroraiq realm
kubectl exec -n dev $KC -- /opt/keycloak/bin/kcadm.sh create users -r auroraiq \
  -s username='admin' -s email='admin@auroraiq.cloud' \
  -s enabled=true -s emailVerified=true

# Set a permanent password (no UPDATE_PASSWORD required action)
kubectl exec -n dev $KC -- /opt/keycloak/bin/kcadm.sh set-password -r auroraiq \
  --username 'admin' --new-password '<your password>'

# Assign the realm role `admin` (NOTE: --uusername, double-u, NOT a typo — it's the
# kcadm short option for "find user by username")
kubectl exec -n dev $KC -- /opt/keycloak/bin/kcadm.sh add-roles -r auroraiq \
  --uusername 'admin' --rolename admin
```

### 9.3 — Verify

```bash
# Should list "admin" in the realm roles assigned to this user
kubectl exec -n dev $KC -- /opt/keycloak/bin/kcadm.sh get-roles \
  -r auroraiq --uusername 'admin' --effective
```

Then on the portal: full **logout → login** as `admin / <password>` → the **Admin** section appears in the menu. If it does not appear, the token isn't carrying the role:
- Make sure you logged out completely (the OIDC session cookie cached the old token).
- In Keycloak, confirm Role mapping shows `admin` (the *realm* role, not a client role).

---

## 10. Phase 9 — Configure Keycloak SMTP (Brevo) from scratch

Keycloak needs an SMTP relay to send signup verify-email, password reset, etc. We use **Brevo** as the relay. The realm's `smtpServer` config holds the host/port/auth + a `from` address that **must be a verified sender** in Brevo, otherwise Brevo silently drops the mail (Keycloak still sees `HTTP 204` because the SMTP handshake succeeds — the drop happens server-side after).

### 10.1 Create a Brevo account (skip if you already have one)

1. Direct signup: **https://app.brevo.com/account/register**
2. Confirm your email, finish onboarding (free plan = 300 mails/day, enough for dev).

### 10.2 Grab the SMTP credentials

Direct page: **https://app.brevo.com/settings/keys/smtp**

You'll see:
- **SMTP server**: `smtp-relay.brevo.com`
- **Port**: `587`
- **Login**: `<id>@smtp-brevo.com` (the *SMTP login*, NOT your Brevo account email)
- **Master password** / SMTP key: starts with `xsmtpsib-...` — click **Generate a new SMTP key** if none is shown, copy it once (Brevo only displays it at creation).

Save both in your password manager.

### 10.3 Add and verify a sender

Direct page: **https://app.brevo.com/senders/list**

1. Click **Ajouter un expéditeur** (top-right).
2. Fill:
   - **From Name**: `AuroraIQ`
   - **From Email**: `admin@auroraiq.cloud` (or any real mailbox you can open on the `auroraiq.cloud` domain).
3. When the popup *"Authentifier votre domaine maintenant ?"* shows → click **Reporter à plus tard**. (Single-sender is enough for dev; full domain auth = clean path for prod.)
4. Open the verification mail Brevo just sent to that address, click the confirm link.
5. Back on the senders page, the row must show a green **Vérifié** badge.

> **Why this matters**: Brevo only relays mails whose `from` matches a verified sender (or a verified domain). Without this, every `testSMTPConnection` returns 204 but recipients never receive anything — Brevo drops silently at the relay.

### 10.4 Configure the `auroraiq` realm SMTP (REST API)

The realm import strategy is `IGNORE_EXISTING`, so editing the `keycloak-realm` ConfigMap won't re-apply on the next pod restart. Use the live REST API instead — the change persists in the realm's DB row.

```bash
# Get an admin access token (master realm)
KC_HOST=http://localhost:30880    # or the NodePort/IngressRoute for Keycloak
KC_ADMIN_PW=$(kubectl -n dev get secret keycloak-secrets \
  -o jsonpath='{.data.KC_BOOTSTRAP_ADMIN_PASSWORD}' | base64 -d)
TOKEN=$(curl -s -X POST "$KC_HOST/realms/master/protocol/openid-connect/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=admin&password=$KC_ADMIN_PW&grant_type=password&client_id=admin-cli" \
  | jq -r .access_token)

# Brevo creds from 10.2 + verified sender from 10.3
BREVO_USER='<id>@smtp-brevo.com'
BREVO_PASS='xsmtpsib-<full-api-key-from-brevo>'
BREVO_FROM='admin@auroraiq.cloud'
BREVO_NAME='AuroraIQ'

# PUT the smtpServer block on the auroraiq realm
curl -s -o /dev/null -w "%{http_code}\n" -X PUT "$KC_HOST/admin/realms/auroraiq" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d "{\"smtpServer\":{
        \"host\":\"smtp-relay.brevo.com\",
        \"port\":\"587\",
        \"from\":\"$BREVO_FROM\",
        \"fromDisplayName\":\"$BREVO_NAME\",
        \"replyTo\":\"\",
        \"starttls\":\"true\",
        \"auth\":\"true\",
        \"ssl\":\"false\",
        \"user\":\"$BREVO_USER\",
        \"password\":\"$BREVO_PASS\"
      }}"
# Expect: 204
```

Verify (password will be masked as `**********`):
```bash
curl -s -H "Authorization: Bearer $TOKEN" "$KC_HOST/admin/realms/auroraiq" \
  | jq '.smtpServer'
```

### 10.5 Test SMTP end-to-end

Keycloak's `testSMTPConnection` sends a real mail to the **currently authenticated admin user's** email. So set an email on the master admin first if it's blank:

```bash
ADMIN_ID=$(curl -s -H "Authorization: Bearer $TOKEN" \
  "$KC_HOST/admin/realms/master/users?username=admin&exact=true" | jq -r '.[0].id')

curl -s -o /dev/null -w "%{http_code}\n" -X PUT \
  "$KC_HOST/admin/realms/master/users/$ADMIN_ID" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"email":"<your-test-inbox@gmail.com>","emailVerified":true}'

# Trigger the test
curl -s -o /dev/null -w "%{http_code}\n" -X POST \
  "$KC_HOST/admin/realms/auroraiq/testSMTPConnection" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "config={\"host\":\"smtp-relay.brevo.com\",\"port\":\"587\",
    \"from\":\"$BREVO_FROM\",\"fromDisplayName\":\"$BREVO_NAME\",\"replyTo\":\"\",
    \"starttls\":\"true\",\"auth\":\"true\",\"ssl\":\"false\",
    \"user\":\"$BREVO_USER\",\"password\":\"$BREVO_PASS\"}"
# Expect: 204  → check the test inbox for "[KEYCLOAK] - SMTP test message"
```

If you get `204` but no mail lands within 2 min:
- **Sender not Vérifié in Brevo** → 10.3 (verify and click confirmation link).
- **Brand-new Brevo account** → https://app.brevo.com/statistics/email → look for the message in Transactional logs to see if Brevo blocked / queued / dropped it.
- **Gmail blackhole** → set up SPF/DKIM/DMARC on `auroraiq.cloud` (domain auth path).

### 10.6 Validate the signup flow

Fresh private window → portal signup → check inbox for the **Verify Email** from `AuroraIQ <admin@auroraiq.cloud>`. Click the link → user lands as verified.