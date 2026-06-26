# k8s-cluster

A GitOps-managed homelab Kubernetes cluster (K3s). Every workload, every config change, every rollout goes through Git and ArgoCD — never a manual `kubectl apply` against the live cluster.

This repo exists as both a working personal infrastructure setup and a sandbox for platform-engineering patterns used in production environments: declarative config, App-of-Apps, an HA control plane, separate ingress tiers for internal vs. public traffic, and Helm/Kustomize side-by-side in the same repo.

---

## Architecture

```
Git (this repo)
   │  watched by
   ▼
ArgoCD  ──  App-of-Apps (ArgoCD/apps/)  ──▶  one child Application per workload
   │
   ▼
K3s cluster — HA control plane via kube-vip
   │
   ├── Traefik (internal)  →  private services   [traefik-internal IngressClass]
   ├── Traefik (external)  →  public-facing apps  [traefik-external IngressClass]
   ├── MetalLB              →  L2 LoadBalancer IPs for both Traefik instances
   ├── cert-manager         →  Let's Encrypt via Cloudflare DNS-01 (no inbound exposure needed)
   ├── Longhorn             →  replicated block storage (SSD tier)
   └── static NFS PVs       →  bulk media/library storage backed by a TrueNAS box
```

Two ingress controllers exist on purpose: `traefik-internal` keeps admin/private tools off the public internet, `traefik-external` is the only path exposed beyond the LAN. Each app's Ingress just picks the class it needs.

---

## How a workload gets deployed

Every entry in `ArgoCD/apps/` is one ArgoCD `Application`, using one of two patterns:

| Pattern | When it's used |
|---|---|
| **Plain Kustomize manifests** | No official Helm chart, or I want full control over every manifest |
| **Multi-source Helm** (upstream chart + values from this repo) | Upstream project ships and maintains a chart |

Adding, removing, or migrating an app between patterns only touches that app's own folder and one line in `ArgoCD/apps/kustomization.yaml` — nothing else in the repo depends on what's currently deployed.

---

## What's running

The cluster hosts a self-hosted alternative to a range of common SaaS tools — media streaming, photo backup, document management, password management, a download/automation stack, a dashboard, and more. The exact, always-current list is the ArgoCD Application set itself:

```bash
ls ArgoCD/apps/
```

This README intentionally doesn't enumerate individual apps — that list changes far more often than the platform underneath it.

---

## Storage

| Tier | Backing | Used for |
|---|---|---|
| `longhorn-ssd` | Replicated (3x) block storage on local SSD | databases, app config — small, latency-sensitive |
| Static NFS PVs | TrueNAS share | media libraries, large bulk data |

## Secrets

Kept out of Git entirely. `secrets/` is gitignored; apps that need a secret ship a placeholder template (`*-secret.yaml`) so the real values can be applied manually post-clone:

```bash
kubectl apply -f secrets/<app>-secret.yaml
```

## Repo layout

```
<App>/                  one folder per workload — manifests or Helm values
ArgoCD/apps/             one Application manifest per workload (the actual deployment graph)
ArgoCD/root-app.yaml     bootstraps everything above — applied once, manually
secrets/                 gitignored Secret templates
clusterissuer-cloudflare.yml
```

---

## Stack

Kubernetes (K3s) · ArgoCD · Helm · Kustomize · Traefik · MetalLB · kube-vip · Longhorn · NFS · cert-manager · Let's Encrypt (Cloudflare DNS-01)
