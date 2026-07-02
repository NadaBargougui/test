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
---
# 1. Install the sealed-secrets controller on the dev-newhs cluster
#    then export its public cert (used to seal everything below):

kubeseal --controller-namespace=kube-system \
         --controller-name=sealed-secrets-controller \
         --fetch-cert > aurora-console/test-managed-services/sealed-secrets/dev-newhs-pub-cert.pem

git push -u origin deploy/dev-newhs

---

> when deploying sealed secret controller in the cluster, it generates one public cret used to encrypt the secrets and one privzte key used to decrypt them! We need to fetch the public key and put it in our repo.


--------------------------------------------------------------------------------------
--------------------------------------------------------------------------------------
NS=dev
OUT=aurora-console/test-managed-services/sealed-secrets/portal
CERT=aurora-console/test-managed-services/sealed-secrets/dev-newhs-pub-cert.pem

# portal-api-secrets
kubectl create secret generic portal-api-secrets -n $NS \
  --from-literal=DB_PASSWORD=<value> \
  --from-literal=INTERNAL_API_TOKEN=<value> \
  --from-literal=KEYCLOAK_CLIENT_ID=<value> \
  --from-literal=KEYCLOAK_ISSUER_URL=https://auth-newhs.auroraiq.cloud/realms/auroraiq \
  --from-literal=PLATFORM_URL=https://portal-newhs.auroraiq.cloud \
  --from-literal=SMTP_HOST=<value> \
  --from-literal=SMTP_LOGIN=<value> \
  --from-literal=SMTP_PASSWORD=<value> \
  --from-literal=SMTP_EMAIL=<value> \
  --from-literal=MGMT_TAILSCALE_KEY=<value> \
  --dry-run=client -o yaml \
  | kubeseal --cert $CERT --format yaml \
  > $OUT/portal-api-secrets.yaml

# portal-db-credentials
kubectl create secret generic portal-db-credentials -n $NS \
  --from-literal=password=<value> \
  --from-literal=postgres-password=<value> \
  --dry-run=client -o yaml \
  | kubeseal --cert $CERT --format yaml \
  > $OUT/portal-db-credentials.yaml

# portal-tls (TLS type, use --from-file with your cert/key files)
kubectl create secret tls portal-tls -n $NS \
  --cert=<path/to/fullchain.pem> \
  --key=<path/to/privkey.pem> \
  --dry-run=client -o yaml \
  | kubeseal --cert $CERT --format yaml \
  > $OUT/portal-tls.yaml

# keycloak-secrets
kubectl create secret generic keycloak-secrets -n $NS \
  --from-literal=KC_BOOTSTRAP_ADMIN_USERNAME=<value> \
  --from-literal=KC_BOOTSTRAP_ADMIN_PASSWORD=<value> \
  --from-literal=KC_DB_USERNAME=<value> \
  --from-literal=KC_DB_PASSWORD=<value> \
  --dry-run=client -o yaml \
  | kubeseal --cert $CERT --format yaml \
  > $OUT/keycloak-secrets.yaml

# ghcr-secret (docker-registry type)
kubectl create secret docker-registry ghcr-secret -n $NS \
  --docker-server=ghcr.io \
  --docker-username=<value> \
  --docker-password=<value> \
  --dry-run=client -o yaml \
  | kubeseal --cert $CERT --format yaml \
  > $OUT/ghcr-secret.yaml

-------------------------  
# aee-cred (file-based bundle)
kubectl create secret generic aee-cred -n $NS \
  --from-file=aee-cred.crt=<path/to/aee-cred.crt> \
  --from-file=client.crt=<path/to/client.crt> \
  --from-file=client.key=<path/to/client.key> \
  --from-file=config.yml=<path/to/config.yml> \
  --dry-run=client -o yaml \
  | kubeseal --cert $CERT --format yaml \
  > $OUT/aee-cred.yaml

------------
create the identity and capture the token
lxc auth identity create tls/aee-runner </dev/null
This prints a one-shot bootstrap token
-------------

On server4:
lxc config trust add --name aee-cred  # generates new token

On portal-01:
# lxc remote add aee-cred https://${MGMT_SUBNET}.1:8443 --token ${TOKEN}
lxc remote add aee-cred https://10.99.0.1:8443 --token <TOKEN>
mkdir -p /tmp/lxd-bundle
cp /root/snap/lxd/common/config/client.crt /tmp/lxd-bundle/
cp /root/snap/lxd/common/config/client.key /tmp/lxd-bundle/
cp /root/snap/lxd/common/config/config.yml /tmp/lxd-bundle/
cp /root/snap/lxd/common/config/servercerts/aee-cred.crt /tmp/lxd-bundle/

for ns in dev aee-runner portal-terminals; do
  case $ns in
    dev)              folder=portal ;;
    aee-runner)       folder=aee-runner ;;
    portal-terminals) folder=portal-terminals ;;
  esac
  kubectl create secret generic aee-cred -n $ns \
    --from-file=client.crt=/tmp/lxd-bundle/client.crt \
    --from-file=client.key=/tmp/lxd-bundle/client.key \
    --from-file=config.yml=/tmp/lxd-bundle/config.yml \
    --from-file=aee-cred.crt=/tmp/lxd-bundle/aee-cred.crt \
    --dry-run=client -o yaml | kubeseal --cert $CERT --format yaml \
    > aurora-console/test-managed-services/sealed-secrets/$folder/aee-cred.yaml

done
-------------------------

# management-ssh-key
kubectl create secret generic management-ssh-key -n $NS \
  --from-file=id_ed25519=<path/to/id_ed25519> \
  --from-file=id_ed25519.pub=<path/to/id_ed25519.pub> \
  --from-file=ssh-private-key=<path/to/id_ed25519> \
  --from-file=ssh-public-key=<path/to/id_ed25519.pub> \
  --dry-run=client -o yaml \
  | kubeseal --cert $CERT --format yaml \
  > $OUT/management-ssh-key.yaml

-----------------------------------------------------------------------------------
NS=argocd
OUT=aurora-console/dev-newhs/sealed-secrets/argocd
CERT=aurora-console/dev-newhs/sealed-secrets/dev-newhs-pub-cert.pem

# argocd-github-secret — needs the app.kubernetes.io/part-of=argocd label
kubectl create secret generic argocd-github-secret -n $NS \
  --from-literal=clientSecret=<value> \
  --dry-run=client -o yaml \
  | kubectl label --local -f - app.kubernetes.io/part-of=argocd -o yaml \
  | kubeseal --cert $CERT --format yaml \
  > $OUT/argocd-github-secret.yaml

# argocd-repo-credentials — needs the argocd.argoproj.io/secret-type=repository label
kubectl create secret generic argocd-repo-credentials -n $NS \
  --from-literal=url=<value> \
  --from-literal=username=<value> \
  --from-literal=password=<value> \
  --from-literal=type=<value> \
  --dry-run=client -o yaml \
  | kubectl label --local -f - argocd.argoproj.io/secret-type=repository -o yaml \
  | kubeseal --cert $CERT --format yaml \
  > $OUT/argocd-repo-credentials.yaml