# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Projects

This repo manages two independent Kubernetes clusters via GitOps (Flux CD):

- **`media-server/`** — single-node k3s cluster for self-hosted media services (primary focus); runs on a Mac Mini M1 (ARM, 8 GB RAM)
- **`ha-cluster/`** — high-availability 3-node k3s cluster across Raspberry Pi 5s (future; currently 2 of 3 nodes available)

Each project is fully self-contained — no cross-project shared manifests.

## Directory Structure

Each project follows this internal layout:

```
<project>/
├── cluster/
│   └── <env>/             # Flux entrypoint — Kustomizations pointing to the rest
├── apps/
│   ├── base/              # Base manifests per app (Deployment, Service, Namespace, PVCs…)
│   └── <env>/             # Environment overlay (patches, config overrides, secrets refs)
├── infrastructure/
│   ├── base/              # Base manifests (cloudflared, renovate…)
│   └── <env>/
└── monitoring/
    ├── base/              # Base manifests (kube-prometheus-stack…)
    └── <env>/
```

`media-server/` currently has a single `staging` environment. Each project manages its own environments independently.

## Toolchain

Tools are managed via [mise](https://mise.jdx.dev/) (`mise.toml`): `kubectl`, `flux2`, `k9s`, `claude-code`.

The devcontainer (`.devcontainer/`) is the standard development environment. It mounts `.kube/` and `.ssh/` from the workspace root into the container home directory and runs with `--network=host` so `kubectl` can reach the cluster directly.

## GitOps Workflow

All cluster state flows exclusively through Git — `kubectl apply` is intentionally avoided.

1. Commit and push to `main`.
2. Flux **Source Controller** polls GitHub (~1 min) and produces an artifact on change.
3. Flux **Kustomize Controller** applies the desired state and auto-remediates any drift.

```bash
# Force immediate reconciliation instead of waiting for the next poll
flux reconcile source git flux-system

# Check reconciliation status
flux get kustomizations
flux get helmreleases -A
```

## Secrets

Secrets are encrypted with [SOPS + age](https://github.com/mozilla/sops) and committed in ciphertext. Only the `data` / `stringData` fields are encrypted, not the full file. The age public key is declared in `<project>/cluster/<env>/.sops.yaml`.

The private age key must exist on the cluster as a Secret named `sops-age` in `flux-system` — Flux uses it to decrypt during reconciliation.

```bash
# Encrypt a secret before committing
sops --encrypt --in-place path/to/secret.yaml
```

## External Access

Services are exposed via a **Cloudflare Tunnel** (`cloudflared`) — no router ports are opened. The tunnel config (hostname → in-cluster service mappings) lives in `infrastructure/base/cloudflared/`. Tunnel credentials are SOPS-encrypted.

## Bootstrap

```bash
# 1. Install k3s
curl -sfL https://get.k3s.io | sh -

# 2. Bootstrap Flux (adjust path for the target project and environment)
flux bootstrap github \
  --owner=remihrt \
  --repository=homelab \
  --branch=main \
  --path=media-server/cluster/staging \
  --personal

# 3. Apply the SOPS decryption key
kubectl create secret generic sops-age \
  --namespace=flux-system \
  --from-file=age.agekey=/path/to/your/key
```
