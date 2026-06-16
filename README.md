# Homelab Journal

A public record of things I broke, fixed, and learned while running a 3-node
Proxmox + k3s Kubernetes cluster.

**Cluster:** 3-node Proxmox (mixed consumer desktops) + k3s v1.35  
**Domain:** [REDACTED]  
**Goal:** Learn platform engineering by operating real services under real
constraints.

## Why This Exists

I already run Immich, Minecraft, Grafana, and ArgoCD in production at home.
This repo documents the incidents not the manifests. The infrastructure
code lives in a private repo. The lesson lives here

## Principles

1. **Immich is sacred.** The `immich` namespace is production. Everything
   else is `crashlab` and exists to be broken.
2. **Postmortems over tutorials.** I learn by inducing failure, not by
   following guides.
3. **Redaction over exposure.** All IPs, tokens, and exact domains are
   scrubbed. The patterns remain.

## Index

| Date | Incident | Severity | Category |
|------|----------|----------|----------|
| 2026-05-08 | ArgoCD Server Args Array Destruction | high | gitops |
| 2026-05-09 | Grafana PVC Permission Denial | medium | storage |
| 2026-05-10 | NGINX Rate Limit Annotation Confusion | low | security |
| 2026-06-01 | NFS-Common Missing on Worker Nodes | high | storage |
| 2026-06-01 | Bitnami Redis Tag Deletion | medium | supply-chain |
| 2026-06-02 | Immich v1→v2 Upgrade + AVX2 Failure | high | application |
| 2026-06-02 | Ingress Missing ingressClassName | medium | networking |
| 2026-06-02 | Cloudflare Bot Fight Mode Mobile Block | low | security |

## Architecture Decisions

- [Why Sealed Secrets over Vault](./architecture-decisions/001-sealed-secrets.md)
- [Why ZFS+NFS over Longhorn for bulk storage](./architecture-decisions/002-zfs-nfs.md)

## Runbooks

- [Recover ArgoCD after bad deployment patch](./runbooks/argocd-recovery.md)
- [Fix PVC permission issues after StorageClass migration](./runbooks/pvc-permissions.md)
