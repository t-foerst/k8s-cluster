# k8s-cluster

GitOps repository for my personal Kubernetes homelab cluster. All workloads are managed declaratively via ArgoCD and deployed to the `foerst.haus` domain with automatic TLS through Let's Encrypt + Cloudflare DNS.

The cluster runs in a **private network and is not reachable from the internet**. DNS resolves the `foerst.haus` subdomains internally only. Cloudflare is used exclusively for the DNS-01 ACME challenge to obtain valid TLS certificates — no traffic is routed through Cloudflare.

---

## Architecture

```
Local Network (private, no public exposure)
    │
    ▼
Traefik (Ingress Controller)
    │
    ├── cert-manager (Let's Encrypt via Cloudflare DNS-01 — certificates only)
    │
    ├── ArgoCD          argocd.foerst.haus
    ├── Longhorn UI     lhorn.foerst.haus
    ├── Homarr          (dashboard)
    ├── Vaultwarden     (passwords)
    ├── Paperless-ngx   paperless.foerst.haus
    ├── Audiobookshelf  (audiobooks)
    ├── Uptime Kuma     (monitoring)
    ├── KI-Coach        (custom AI app)
    └── MinIO           (object storage)
```

---

## Infrastructure

| Component | Purpose |
|---|---|
| **ArgoCD** | GitOps continuous deployment — watches this repo and applies changes |
| **Longhorn** | Distributed block storage with SSD and HDD storage classes (3 replicas each) |
| **Traefik** | Ingress controller — routes external traffic into the cluster |
| **cert-manager** | Automatic TLS certificates via Let's Encrypt (Cloudflare DNS-01 challenge) |
| **Headlamp** | Web-based Kubernetes admin panel |

---

## Services

| Service | Description | Stack |
|---|---|---|
| **Vaultwarden** | Self-hosted Bitwarden-compatible password manager | `vaultwarden/server` |
| **Homarr** | Home dashboard for all services | `homarr-labs/homarr` |
| **Paperless-ngx** | Document management system | `paperless-ngx` + PostgreSQL + Redis |
| **Audiobookshelf** | Audiobook & podcast server | `advplyr/audiobookshelf` |
| **Uptime Kuma** | Uptime monitoring and status pages | `louislam/uptime-kuma` |
| **KI-Coach** | Custom AI coaching app (Gemini-powered) | React frontend + Python backend |
| **MinIO** | S3-compatible object storage | `minio/minio` |

---

## Repository Structure

```
k8s-cluster/
│
├── ArgoCD/                  # Ingress for ArgoCD UI
├── Admin-Panel/             # Headlamp ServiceAccount + ClusterRoleBinding
├── Longhorn/                # Storage classes (SSD/HDD) + Ingress
│
├── Audiobookshelf/          # Deployment, Service, Ingress, PVCs
├── Homarr/                  # Deployment, Service, Ingress, PVC
├── Paperless-ngx/           # Paperless + PostgreSQL + Redis
├── UptimeKuma/              # Deployment, Service, Ingress, PVC
├── Vaultwarden/             # Deployment, Service, Ingress, PVC
│
├── ki-coach-backend/        # Python/FastAPI backend (Gemini API)
├── ki-coach-frontend/       # React frontend
│
├── secrets/                 # Kubernetes Secret manifests (not committed in plain text)
│
└── clusterissuer-cloudflare.yml   # cert-manager ClusterIssuer (Let's Encrypt + Cloudflare)
```

Each application directory follows this pattern:
- `namespace.yaml` — dedicated namespace
- `deployment.yaml` — workload definition
- `service.yaml` — internal ClusterIP service
- `ingress.yaml` — Traefik ingress with TLS
- `pvc-*.yaml` — PersistentVolumeClaims (backed by Longhorn)
- `kustomization.yaml` — Kustomize manifest list

---

## Storage

Longhorn provides two storage classes:

| StorageClass | Disk | Replicas |
|---|---|---|
| `longhorn-ssd` | SSD (diskSelector: `ssd`) | 3 |
| `longhorn-hdd` | HDD (diskSelector: `hdd`) | 3 |

---

## TLS / Certificates

All ingresses use the `letsencrypt-dns` ClusterIssuer, which solves the ACME DNS-01 challenge via a Cloudflare API token. This means valid Let's Encrypt certificates are issued without any inbound internet access — the cluster only needs outbound HTTPS to reach the Let's Encrypt and Cloudflare APIs. Certificates are requested automatically when an Ingress with the annotation `cert-manager.io/cluster-issuer: letsencrypt-dns` is created.

---

## Secrets

Secrets are stored as Kubernetes Secret manifests in the `secrets/` directory. These contain sensitive values and are **not committed in plain text** — they must be applied manually or managed via a secrets solution (e.g. Sealed Secrets, External Secrets Operator).

```bash
kubectl apply -f secrets/
```

---

## Applying Manifests

Individual apps can be applied with Kustomize:

```bash
kubectl apply -k Paperless-ngx/
kubectl apply -k Vaultwarden/
kubectl apply -k Homarr/
# etc.
```

Or let ArgoCD handle it automatically by pointing an Application resource at this repository.
