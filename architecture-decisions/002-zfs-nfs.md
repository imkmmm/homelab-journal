---
date: 2026-05-08
status: accepted
---

# ADR-002: ZFS + NFS over Longhorn for Bulk Storage

## Context

Only one node (Proxmox 2, OptiPlex 7070) has an 8TB HDD. The other two
nodes have only SSDs (~200-500GB). Immich needs ~2TB for photo storage.

## Options Considered

| Option | Pros | Cons |
|--------|------|------|
| **Longhorn** | Distributed, replicated, self-healing | Would fail (can't replicate 8TB to 200GB nodes) or waste space |
| **ZFS + NFS** | Uses existing 8TB, compression, snapshots | Single point of failure (one node), not replicated |
| **Ceph** | Distributed, scalable | Massive overhead, needs 3+ nodes with similar storage |

## Decision

Use **ZFS on Proxmox 2** with **NFS export** to Kubernetes.

## Consequences

- Photo storage is not replicated across nodes (acceptable risk for homelab)
- ZFS snapshots provide point-in-time recovery
- NFS requires `nfs-common` on all worker nodes (Day-0 dependency)
- Postgres database stays on `local-path` SSD (never put databases on NFS)

## Status

Accepted. Will re-evaluate if a second large drive is added for ZFS mirror.
