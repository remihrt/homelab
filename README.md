# 🏠 Homelab

> A production-grade Kubernetes homelab running on a heterogeneous Raspberry Pi + x86 cluster, managed with GitOps using Flux CD.
>
> Built for hands-on, enterprise-grade learning — and as a living portfolio for the CKA certification path.

![Kubernetes](https://img.shields.io/badge/Kubernetes-k3s-326CE5?logo=kubernetes&logoColor=white)
![Flux](https://img.shields.io/badge/GitOps-Flux_CD-5468FF?logo=flux&logoColor=white)
![Renovate](https://img.shields.io/badge/Dependencies-Renovate-1A1F6C?logo=renovatebot&logoColor=white)
![SOPS](https://img.shields.io/badge/Secrets-SOPS-FF6B35)
![License](https://img.shields.io/badge/License-MIT-green)

---

## 📖 Overview

This repository contains the full GitOps configuration for my personal Kubernetes homelab. Every resource — from infrastructure components to self-hosted applications — is declared as code and continuously reconciled by Flux CD.

The goals of this project are to:

- Practice enterprise-grade Kubernetes patterns in a real, physical environment
- Build a self-hosted platform for personal services with a focus on reliability and security
- Develop the skills required to pass the **Certified Kubernetes Administrator (CKA)** certification
- Serve as a concrete, inspectable portfolio for technical recruiters

---

## 🖥️ Hardware

The cluster runs on a heterogeneous mix of ARM and x86 nodes, which adds real-world complexity around multi-architecture image compatibility.

| Role          | Host                          | Architecture | CPU        | RAM   |
|---------------|-------------------------------|--------------|------------|-------|
| Control Plane | Raspberry Pi 5                | ARM64        | Cortex-A76 | 4 GB  |
| Worker        | Raspberry Pi 5                | ARM64        | Cortex-A76 | 4 GB  |
| Worker        | MacBook Pro (Arch Linux)      | x86_64       | Intel Core | 16 GB |

---

## ⚙️ Tech Stack

| Layer              | Tool                  | Purpose                                        |
|--------------------|-----------------------|------------------------------------------------|
| Kubernetes         | k3s                   | Lightweight, production-ready K8s distribution |
| GitOps             | Flux CD               | Continuous reconciliation from Git             |
| Ingress            | Traefik               | Reverse proxy and TLS termination              |
| Observability      | Prometheus + Grafana  | Metrics collection and dashboarding            |
| Secrets            | SOPS                  | Encrypted secrets committed safely to Git      |
| Dependency updates | Renovate              | Automated image and chart version bumping      |
| Tunnel             | Cloudflared           | Secure external access via Cloudflare Tunnel   |

---

## 📁 Repository Structure

```
homelab/
├── apps/                    # Self-hosted applications
├── clusters/
│   └── staging/             # Cluster entrypoint — tells Flux what to reconcile
├── flux-system/             # Flux controllers, generated during bootstrap
├── infrastructure/          # Platform-level services (Renovate, Cloudflared)
├── monitoring/              # Observability stack (Prometheus, Grafana)
├── renovate.json            # Renovate bot configuration
└── .gitignore
```

### Why this structure?

The `clusters/staging/` directory is the **entrypoint**: it contains Flux `Kustomization` objects that point to every other directory. Flux reads this entrypoint first, then fans out to reconcile `apps/`, `infrastructure/`, and `monitoring/` independently. This separation of concerns mirrors patterns used in production GitOps platforms.

---

## 🔄 GitOps Workflow

All changes to the cluster flow exclusively through Git. Direct `kubectl apply` commands are intentionally avoided.

```
┌─────────────────────────────────────────────────────────────┐
│                        GitOps Flow                          │
│                                                             │
│  git push ──► GitHub ──► Source Controller (every ~1 min)  │
│                                │                            │
│                         Fetches repo &                      │
│                         produces artifact                   │
│                                │                            │
│                         Kustomize Controller                │
│                                │                            │
│                    Applies desired state to cluster         │
│                                │                            │
│              Continuous drift remediation (auto-revert)     │
└─────────────────────────────────────────────────────────────┘
```

1. A commit is pushed to the `main` branch on GitHub.
2. **Source Controller** fetches the repository on a regular interval and produces a local artifact when a change is detected.
3. **Kustomize Controller** reads the artifact, computes the desired state, and applies it to the cluster.
4. If manual changes are made directly to the cluster (drift), Kustomize Controller automatically remediates them by re-applying the Git state.

> To trigger an immediate reconciliation without waiting for the next interval:
> ```bash
> flux reconcile source git flux-system
> ```

---

## 🔐 Secrets Management

Secrets are encrypted at rest using **SOPS** and committed directly to this repository. They are never stored in plaintext.

The decryption key is managed separately and configured on the cluster, allowing Flux to decrypt secrets automatically during reconciliation. This pattern is safe for public repositories.

---

## 🚀 Bootstrap

> ⚠️ Prerequisites: `kubectl`, `flux` CLI, and a SOPS key configured locally.

```bash
# 1. Install k3s on the control plane node
curl -sfL https://get.k3s.io | sh -

# 2. Bootstrap Flux on the cluster, pointing to this repository
flux bootstrap github \
  --owner=remihrt \
  --repository=homelab \
  --branch=main \
  --path=clusters/staging \
  --personal

# 3. Apply SOPS decryption secret
kubectl create secret generic sops-age \
  --namespace=flux-system \
  --from-file=age.agekey=/path/to/your/key
```

After bootstrap, Flux will automatically reconcile the full stack.

---

## 🗺️ Roadmap

- [ ] Add multi-node high availability for the control plane
- [ ] Implement network policies for inter-namespace traffic control
- [ ] Add automated alerting via Alertmanager
- [ ] Pass the CKA certification 🎯

---

## 🙏 Acknowledgements

Inspired by the incredible homelab and GitOps community. Key references:

- [Flux CD Documentation](https://fluxcd.io/docs/)
- [k3s Documentation](https://docs.k3s.io/)
- [awesome-home-kubernetes](https://github.com/k8s-at-home/awesome-home-kubernetes)
- [SOPS by Mozilla](https://github.com/mozilla/sops)
