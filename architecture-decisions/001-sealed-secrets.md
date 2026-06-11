---
date: 2026-05-07
status: accepted
---

# ADR-001: Sealed Secrets over HashiCorp Vault

## Context

The homelab uses GitOps (ArgoCD) with a private GitHub repository. All
Kubernetes manifests are stored in Git. Secrets must be stored alongside
manifests without exposing plaintext.

## Options Considered

| Option | Pros | Cons |
|--------|------|------|
| **Sealed Secrets** | Zero external deps, encrypted blobs safe in any repo | Key loss = all secrets lost |
| **HashiCorp Vault** | Full-featured, dynamic secrets, ACLs | Needs dedicated VM, complex, overkill for 3-node lab |
| **Mozilla SOPS** | Flexible encryption | Needs cloud KMS or age keys; homelab has no cloud KMS |
| **External Secrets Operator** | Syncs from cloud secret managers | Requires cloud account + billing |
| **Native Kubernetes Secrets** | Built-in | Base64, not encrypted, unsafe in Git |

## Decision

Use **Bitnami Sealed Secrets**.

## Consequences

- Master key backed up to KeePassXC immediately after installation
- Local backup file shredded with `shred -u` and bash history cleared
- Classic PAT used for ArgoCD private repo access (Fine-Grained PAT had GitHub UI bugs)

## Status

Accepted. No plans to migrate unless cluster grows to multi-team or requires dynamic secrets.
