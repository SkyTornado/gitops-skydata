
[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](https://www.gnu.org/licenses/gpl-3.0)

# Kubernetes GitOps Stack Setup

This README describes how to manually install and configure the following components in a GKE cluster with Workload Identity, **and** how to push selected Kubernetes Secrets into Google Secret Manager:

1. Ingress‑NGINX  
2. External‑DNS  
3. External‑Secrets Operator  
4. Cert‑Manager v1.17.1 (with custom Google DNS resolvers)  
5. Let’s Encrypt **Production** ClusterIssuer  
6. Argo CD v7.8.27 (with Ingress + Production TLS)  
7. **(new!)** Push selected K8s Secrets into GCP Secret Manager via ClusterPushSecret  

---

## 🔧 Prerequisites

- A GKE cluster created via Terragrunt; `kubectl` is pointed at it.  
- **Workload Identity** linking these GCP ServiceAccounts ↔ K8s ServiceAccounts:

  | Component         | GCP SA                                                                    | K8s SA                                     | Namespace           |
  |-------------------|----------------------------------------------------------------------------|---------------------------------------------|---------------------|
  | external‑dns      | `external-dns-stage-skydata-gke@skydata-379820.iam.gserviceaccount.com`    | `external-dns-stage-skydata-gke-main`      | `external-dns`      |
  | external‑secrets  | `external-secrets-stage-skydata@skydata-379820.iam.gserviceaccount.com`    | `external-secrets-stage-skydata-gke-main`  | `external-secrets`  |
  | cert‑manager      | `cert-manager-stage-skydata-gke@skydata-379820.iam.gserviceaccount.com`    | `cert-manager-stage-skydata-gke-main`      | `cert-manager`      |

- The GCP SAs above have at least:
  - `roles/iam.serviceAccountTokenCreator`  
  - **for pushing into Secret Manager:** `roles/secretmanager.admin`  

---

## 1. Install Ingress‑NGINX

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
kubectl create namespace ingress-nginx

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --set controller.service.type=LoadBalancer \
  --set controller.service.annotations."external-dns\.alpha\.kubernetes\.io/hostname"="skydata.space" \
  --set controller.ingressClassResource.name=nginx \
  --set controller.ingressClassResource.controllerValue="k8s.io/ingress-nginx"

kubectl get pods,svc -n ingress-nginx
```

---

## 2. Install External‑DNS

```bash
helm repo add external-dns https://kubernetes-sigs.github.io/external-dns
helm repo update
kubectl create namespace external-dns

# Verify K8s SA annotation
kubectl get sa external-dns-stage-skydata-gke-main -n external-dns -o yaml

# Grant token‑creator on the GCP SA if not already:
gcloud iam service-accounts add-iam-policy-binding \
  external-dns-stage-skydata-gke@skydata-379820.iam.gserviceaccount.com \
  --member="serviceAccount:skydata-379820.svc.id.goog[external-dns/external-dns-stage-skydata-gke-main]" \
  --role="roles/iam.serviceAccountTokenCreator"

helm install external-dns external-dns/external-dns \
  --namespace external-dns \
  --set provider=google \
  --set google.project=skydata-379820 \
  --set serviceAccount.name=external-dns-stage-skydata-gke-main \
  --set serviceAccount.create=false \
  --set domainFilters={skydata.space} \
  --set policy=sync \
  --set txtOwnerId=skydata

kubectl get pods -n external-dns
kubectl logs -n external-dns -l app.kubernetes.io/name=external-dns
```

---

## 3. Install External‑Secrets Operator

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm repo update
kubectl create namespace external-secrets

# Verify K8s SA annotation
kubectl get sa external-secrets-stage-skydata-gke-main -n external-secrets -o yaml

# Grant token‑creator on the GCP SA:
gcloud iam service-accounts add-iam-policy-binding \
  external-secrets-stage-skydata@skydata-379820.iam.gserviceaccount.com \
  --member="serviceAccount:skydata-379820.svc.id.goog[external-secrets/external-secrets-stage-skydata-gke-main]" \
  --role="roles/iam.serviceAccountTokenCreator"

# And Secret Manager Admin so we can push secrets later:
gcloud projects add-iam-policy-binding skydata-379820 \
  --member="serviceAccount:external-secrets-stage-skydata@skydata-379820.iam.gserviceaccount.com" \
  --role="roles/secretmanager.admin"

# Enable automounting of the Workload Identity token
kubectl patch sa external-secrets-stage-skydata-gke-main -n external-secrets \
  -p '{"automountServiceAccountToken":true}'

helm install external-secrets external-secrets/external-secrets \
  --namespace external-secrets \
  --set installCRDs=true \
  --set provider.gcpSecretsManager.projectID=skydata-379820 \
  --set serviceAccount.name=external-secrets-stage-skydata-gke-main \
  --set serviceAccount.create=false \
  --set rbac.create=true

kubectl get pods -n external-secrets
kubectl logs -n external-secrets -l app.kubernetes.io/name=external-secrets
```

---

## 4. Install Cert‑Manager v1.17.1 with Custom DNS Resolvers

```bash
# 4.1 Cleanup old CRDs (if any):
kubectl delete crd certificaterequests.cert-manager.io certificates.cert-manager.io \
  challenges.acme.cert-manager.io clusterissuers.cert-manager.io \
  issuers.cert-manager.io orders.acme.cert-manager.io || true

# 4.2 Apply v1.17.1 CRDs:
kubectl apply --validate=false \
  -f https://github.com/jetstack/cert-manager/releases/download/v1.17.1/cert-manager.crds.yaml

# 4.3 Patch K8s SA:
kubectl patch sa cert-manager-stage-skydata-gke-main -n cert-manager \
  -p '{"automountServiceAccountToken":true}'

# 4.4 Install chart:
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm upgrade --install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v1.17.1 \
  --set installCRDs=false \
  --set serviceAccount.name=cert-manager-stage-skydata-gke-main \
  --set serviceAccount.create=false

# 4.5 Configure Google DNS resolvers:
kubectl edit deployment cert-manager -n cert-manager
```

_Add under_ `spec.template.spec.containers[0].args:`:
```yaml
- --dns01-recursive-nameservers=ns-cloud-c1.googledomains.com:53,ns-cloud-c2.googledomains.com:53,ns-cloud-c3.googledomains.com:53,ns-cloud-c4.googledomains.com:53
```

```bash
kubectl rollout status deployment cert-manager -n cert-manager
kubectl get pods -n cert-manager
```

---

## 5. Create Let’s Encrypt Production ClusterIssuer

```bash
cat <<EOF > letsencrypt-production-issuer.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-production
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: you@skydata.space
    privateKeySecretRef:
      name: le-production-account-key
    solvers:
      - dns01:
          cloudDNS:
            project: skydata-379820
EOF

kubectl apply -f letsencrypt-production-issuer.yaml
kubectl wait --for=condition=Ready clusterissuer/letsencrypt-production --timeout=120s
kubectl get clusterissuer letsencrypt-production -o wide
```

---

## 6. Install Argo CD v7.8.27 with Ingress & Production TLS

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
kubectl create namespace argocd

cat <<EOF > argocd-values.yaml
server:
  ingress:
    enabled: true
    ingressClassName: nginx
    hostname: argo.skydata.space
    annotations:
      external-dns.alpha.kubernetes.io/hostname: argo.skydata.space
      nginx.ingress.kubernetes.io/backend-protocol: HTTPS
      nginx.ingress.kubernetes.io/ssl-passthrough: "true"
      cert-manager.io/cluster-issuer: letsencrypt-production
    hosts:
      - argo.skydata.space
    tls:
      - hosts:
          - argo.skydata.space
        secretName: argocd-server-tls
  configs:
    params:
      server.insecure: false
EOF

helm install argocd argo/argo-cd \
  --namespace argocd \
  --version 7.8.27 \
  --values argocd-values.yaml

kubectl rollout status deployment argocd-server -n argocd
kubectl get ingress -n argocd argocd-server -o wide
kubectl describe certificate argocd-server-tls -n argocd
```

---

## 7. Push Selected K8s Secrets into Google Secret Manager

### 7.1 Create a ClusterSecretStore

```bash
kubectl delete clustersecretstore gcp-secretstore 2>/dev/null || true

kubectl apply -f - <<EOF
apiVersion: external-secrets.io/v1alpha1
kind: ClusterSecretStore
metadata:
  name: gcp-secretstore
spec:
  provider:
    gcpSecretsManager:
      projectID: skydata-379820
EOF
```

### 7.2 Push the ACME account key

```bash
cat <<EOF > push-le-production-key.yaml
apiVersion: external-secrets.io/v1alpha1
kind: ClusterPushSecret
metadata:
  name: push-le-production-key
spec:
  pushSecretSpec:
    secretStoreRefs:
      - name: gcp-secretstore
        kind: ClusterSecretStore
    selector:
      secret:
        name: le-production-account-key
    data:
      - match:
          secretKey: tls.key
          remoteRef:
            remoteKey: le-production-account-key
EOF

kubectl apply -f push-le-production-key.yaml
kubectl describe clusterpushsecret push-le-production-key
```

### 7.3 Push the Argo CD TLS secret

```bash
cat <<EOF > push-argocd-server-tls.yaml
apiVersion: external-secrets.io/v1alpha1
kind: ClusterPushSecret
metadata:
  name: push-argocd-server-tls
spec:
  pushSecretSpec:
    secretStoreRefs:
      - name: gcp-secretstore
        kind: ClusterSecretStore
    selector:
      secret:
        name: argocd-server-tls
    data:
      - match:
          secretKey: tls.crt
          remoteRef:
            remoteKey: argocd-server-crt
      - match:
          secretKey: tls.key
          remoteRef:
            remoteKey: argocd-server-key
EOF

kubectl apply -f push-argocd-server-tls.yaml
kubectl describe clusterpushsecret push-argocd-server-tls
```

> **Note:** granting `roles/secretmanager.admin` to the GCP SA allows ESO to **create** new secrets and **add versions**. If you only need to read or update existing secrets, consider `roles/secretmanager.secretAccessor` + `roles/secretmanager.secretVersionAdder`.

---

## ✅ Next Steps

1. **Retrieve the Argo CD admin password**:
   ```bash
   kubectl -n argocd get secret argocd-initial-admin-secret \
     -o go-template="{{ .data.password | base64decode }}"
   ```
2. **Delete the initial admin secret** as recommended by Argo CD docs.
3. **Define your Applications & Projects** in Argo CD for GitOps.

## License

GNU GPL v3
```