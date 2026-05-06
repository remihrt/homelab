# Homelab

GitOps-managed Kubernetes homelab using [Flux CD](https://fluxcd.io/) and [k3s](https://k3s.io/).

## Projects

### `media-server/`

Single-node k3s cluster running self-hosted media services. Managed entirely through this repo — no direct `kubectl apply`.

**Hardware:** Mac Mini M1 (ARM, 8 GB RAM, 128 GB SSD) 

### `ha-cluster/`

High-availability 3-node k3s cluster. Work in progress — currently 2 of 3 Raspberry Pi 5s available.

**Hardware:** 3× Raspberry Pi 5 (ARM64, 4 GB RAM, 64 GB USB)

---

## Repository structure

```
homelab/
├── media-server/
│   └── cluster/staging/       # Flux entrypoint
└── ha-cluster/
    ├── cluster/staging/       # Flux entrypoint
    └── apps/
        ├── base/linkding/
        └── staging/linkding/
```

Each project is self-contained. Within a project, manifests follow a `base/` + environment overlay pattern (e.g. `staging/`).

## Stack

| Layer       | Tool                  |
|-------------|-----------------------|
| Kubernetes  | k3s                   |
| GitOps      | Flux CD               |
| Secrets     | SOPS + age            |
| Updates     | Renovate              |
| Tunnel      | Cloudflare Tunnel     |
| Observability | Prometheus + Grafana |

## Bootstrap

```bash
# Install k3s
curl -sfL https://get.k3s.io | sh -

# Bootstrap Flux (adjust path for the target project)
flux bootstrap github \
  --owner=remihrt \
  --repository=homelab \
  --branch=main \
  --path=media-server/cluster/staging \
  --personal

# Apply the SOPS decryption key
kubectl create secret generic sops-age \
  --namespace=flux-system \
  --from-file=age.agekey=/path/to/your/key
```

## Secrets

Secrets are encrypted with [SOPS](https://github.com/mozilla/sops) (age) before being committed. Only the `data` / `stringData` fields are encrypted. The age public key for each project is in `<project>/cluster/<env>/.sops.yaml`.

```bash
sops --encrypt --in-place path/to/secret.yaml
```

## Development

Tools are managed via [mise](https://mise.jdx.dev/): `kubectl`, `flux2`, `k9s`. A devcontainer is provided at `.devcontainer/`.

```bash
# Force immediate Flux reconciliation
flux reconcile source git flux-system

# Check status
flux get kustomizations
flux get helmreleases -A
```
