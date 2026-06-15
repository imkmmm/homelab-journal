---
date: 2026-06-01
severity: high
category: application
affected: immich-server, immich-ml
status: resolved
---

# Immich v1 to v2 Upgrade and AVX2 Incompatibility

## Summary

Upgraded Immich from v1 to v2. Server crashlooped with Invalid upgrade path. After wiping the database and starting fresh, the machine-learning pod crashlooped with a NumPy runtime error about baseline optimizations.

## Impact

- Immich completely unavailable during troubleshooting
- Risk of data loss if database wipe had been attempted on populated instance
- Duration: ~1 hour

## Timeline

- **09:00** Updated ArgoCD Application to Immich chart v2
- **09:10** Server crashloop: Invalid upgrade path
- **09:30** Decision: instance was new, wiped database
- **09:40** Deleted Postgres StatefulSet and PVC; let ArgoCD recreate
- **09:50** Server started successfully
- **10:00** Machine-learning pod crashloop: NumPy AVX2 error
- **10:15** Disabled machine-learning component entirely
- **10:20** All pods stable

## Root Cause

Two separate factors:

1. Database migration: Immich v2 uses a different schema. The application does not auto-migrate. It requires manual pg_dump/restore or a dedicated migration tool.

2. AVX2 CPU feature mismatch: The ML container uses NumPy and ONNX Runtime compiled with AVX2 baseline. The container's baseline detection failed at runtime, likely due to virtualized /proc/cpuinfo.

## Resolution

For the database (acceptable because instance had no user data):

```bash
kubectl scale deployment immich-server -n immich --replicas=0
kubectl delete statefulset immich-postgres -n immich
kubectl delete pvc data-immich-postgres-0 -n immich
kubectl scale deployment immich-server -n immich --replicas=1
```

For the ML component, it is disabled in Helm values: 

```yaml
machine-learning: 
  enabled: false
```

##Lessons Learned 

1. Major version upgrades are not just changing the tag
2. Wiping a databse is only acceptable when verified that no data exists.
3. ML workloads are sensitive to hardware. Container image must mach host CPU features.
4. Feature flags are important. It lets know to disable ML, which allowed core app to funciton.

##Action Items

- [x] Document Immich upgrade procedure
- [x] Disable ML in production values
- [ ] Set up staging namespace for testing major version jumps
- [ ] Evaluate lighter ML model for CPU-only deployment

##Supporting Evidence

Server crashloop log:
```
Invalid upgrade path. The database schema is from v1.x.
Please follow the upgrade guide at https://docs.immich.app/errors
```
ML crashloop log: 
```
RuntimeError: NumPy was built with baseline optimizations: (X86_V2)
and cannot run on this CPU.
```

##Note on Data Safety 

If this had occcurred on a populated instance, the correct resolution would have been:

1. pg_dump the existing database
2. Follow official v1 to v2 migration guide
3. Restore into new Postgres v2-compatible schema
4. Never delete PVC before verifying that the  dump is valid

