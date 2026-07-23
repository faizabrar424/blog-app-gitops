# blog-app-gitops

Manifest Kubernetes (GitOps) untuk aplikasi blog-app (Next.js + MongoDB) yang dikelola menggunakan Argo CD pada cluster k3s. Repository ini menyimpan seluruh konfigurasi aplikasi, mulai dari workload, Gateway API, TLS, RBAC, hingga NetworkPolicy.

> Source code aplikasi (Next.js + Dockerfile + CI build) ada di repo terpisah: **`blog-app`**.

## Architecture

```
                              Client
                                |
                                v
                  Envoy Gateway (Gateway API)
                  +- :80  HTTP  -> redirect to HTTPS
                  +- :443 HTTPS -> TLS terminated (cert-manager)
                                |
                                v
                     Service: blog-app (ClusterIP)
                                |
                                v
                Deployment: blog-app (Next.js)
                +- HPA - CPU-based, 1-3 replicas
                +- Liveness/Readiness - /api/health
                +- NetworkPolicy - ingress restricted to Gateway namespace
                                |
                                v
                     Service: mongodb (ClusterIP)
                                |
                                v
          StatefulSet: mongodb - Replica Set, 3 nodes
          +- mongodb-0 (Primary)
          +- mongodb-1 (Secondary)
          +- mongodb-2 (Secondary)
          Headless Service for pod-to-pod discovery
          One PVC per pod (volumeClaimTemplates)
```

## Tech Stack

| Layer             | Teknologi                                    |
| ----------------- | -------------------------------------------- |
| Orkestrasi        | k3s                                          |
| App runtime       | Next.js (blog-app)                           |
| Database          | MongoDB 7 — Replica Set, StatefulSet 3 node  |
| Traffic / Ingress | Envoy Gateway (Gateway API)                  |
| TLS               | cert-manager (CA Issuer; siap ACME)          |
| Metrics           | Prometheus (kube-prometheus-stack) + Grafana |
| Logs              | Loki + Grafana Alloy                         |
| CI                | GitHub Actions                               |
| CD                | ArgoCD (GitOps, pull-based)                  |

## Features

- GitOps deployment menggunakan Argo CD
- MongoDB Replica Set untuk high availability
- HTTPS dengan Gateway API dan cert-manager
- Horizontal Pod Autoscaler
- Monitoring dengan Prometheus dan Grafana
- Centralized logging dengan Loki dan Alloy
- NetworkPolicy dan RBAC (ServiceAccount minimal per komponen)

## Deployment Workflow

```
git tag v1.0.x → push (repo blog-app)
        │
        ▼
GitHub Actions: build image → push ke GHCR
        │
        ▼
GitHub Actions: clone repo ini → update image tag → commit → push
        │
        ▼
Argo CD mendeteksi perubahan → auto-sync
        │
        ▼
Rolling update Pod ke image baru
```

Dalam pendekatan GitOps, repository menjadi acuan konfigurasi cluster. Perubahan dilakukan melalui commit ke repository, sedangkan perubahan manual akan dikembalikan oleh Argo CD (selfHeal: true) agar tetap sesuai dengan konfigurasi di Git.

## Repository Structure

```
blog-app-gitops/
├── namespace.yaml              # namespace `blog-app`
│
├── argocd/
│   └── application.yaml        # Argo CD Application — recurse:true, prune, selfHeal
│
├── blog-app/
│   ├── deployment.yaml         # probes, resource limits, serviceAccountName
│   ├── service.yaml            # ClusterIP :3000
│   ├── configmap.yaml          # env non-sensitif (DB name, timezone, base URL)
│   ├── secret.yaml             # not committed — see .gitignore
│   └── hpa.yaml                # CPU 70%, 1-3 replica, anti-thrashing behavior
│
├── mongodb/
│   ├── deployment.yaml         # StatefulSet, replSet rs0, 3 replica
│   ├── service.yaml            # ClusterIP :27017 (load-balanced, dipakai blog-app)
│   ├── headless-service.yaml   # clusterIP: None — sinkronisasi antar-pod
│   └── secret.yaml             # not committed — see .gitignore
│
├── gateway/
│   ├── gateway.yaml             # listener HTTP:80, HTTPS:443
│   ├── httproute.yaml           # route HTTPS -> Service blog-app
│   └── http-redirect.yaml       # redirect HTTP -> HTTPS
│
├── ingress.yaml                 # referensi historis (sebelum migrasi ke Gateway API)
│
├── ca-issuer/
│   ├── cluster-issuer.yaml      # ClusterIssuer, CA internal
│   ├── ca.crt                   # not committed — see .gitignore
│   └── ca.key                   # not committed — see .gitignore
│
├── rbac/
│   ├── serviceaccount.yaml          # blog-app-sa, automountServiceAccountToken: false
│   └── serviceaccount-mongodb.yaml  # mongodb-sa
│
├── network-policy/
│   ├── blog-app-netpol.yaml     # ingress dari gateway saja, egress ke mongodb + DNS
│   └── mongodb-netpol.yaml      # ingress dari blog-app + sesama pod mongodb saja
│
└── .gitignore                   # secret.yaml, ca.key, ca.crt
```

## Getting Started

### Prerequisites

Pastikan sudah terpasang di cluster sebelum lanjut: **k3s**, **Helm**, **`kubectl`**, dan **Metrics Server** aktif (dibutuhkan untuk HPA).

```bash
# cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml

# Envoy Gateway (Gateway API)
helm install eg oci://docker.io/envoyproxy/gateway-helm \
  --version v1.8.2 -n envoy-gateway-system --create-namespace

# Argo CD
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 1. Clone repository

```bash
git clone https://github.com/<user>/blog-app-gitops.git
cd blog-app-gitops
```

### 2. Buat Secret & CA (tidak disimpan di Git)

```bash
kubectl create secret generic blog-app-secret \
  --namespace blog-app \
  --from-literal=MONGOLOQUENT_DATABASE_URI="mongodb://<user>:<pass>@mongodb-0.mongodb-headless...:27017,mongodb-1...:27017,mongodb-2...:27017/?authSource=admin&replicaSet=rs0"

kubectl create secret generic mongodb-secret \
  --namespace blog-app \
  --from-literal=MONGO_INITDB_ROOT_USERNAME=<user> \
  --from-literal=MONGO_INITDB_ROOT_PASSWORD=<pass>

# CA untuk TLS internal
openssl genrsa -out ca.key 2048
openssl req -x509 -new -nodes -key ca.key -subj "/CN=blog-app-ca" -days 3650 -out ca.crt
kubectl create secret tls blog-app-ca --namespace cert-manager --cert=ca.crt --key=ca.key
```

### 3. Deploy via Argo CD

```bash
kubectl apply -f argocd/application.yaml
```

Argo CD otomatis membuat namespace, melakukan sync rekursif seluruh manifest (`directory.recurse: true`), dengan `prune` dan `selfHeal` aktif.

### 4. Inisialisasi MongoDB Replica Set (sekali saja)

```bash
kubectl exec -it mongodb-0 -n blog-app -- mongosh -u <user> -p <pass> --authenticationDatabase admin
```

```js
rs.initiate({
  _id: 'rs0',
  members: [
    { _id: 0, host: 'mongodb-0.mongodb-headless.blog-app.svc.cluster.local:27017' },
    { _id: 1, host: 'mongodb-1.mongodb-headless.blog-app.svc.cluster.local:27017' },
    { _id: 2, host: 'mongodb-2.mongodb-headless.blog-app.svc.cluster.local:27017' },
  ],
});
```

### 5. Verifikasi & Akses

```bash
kubectl get application blog-app -n argocd
kubectl get pods -n blog-app
```

Akses aplikasi di `https://blog-app.labstack` (arahkan DNS/`/etc/hosts` ke IP eksternal Envoy Gateway, import `ca.crt` ke browser untuk menghilangkan warning TLS).

## License
