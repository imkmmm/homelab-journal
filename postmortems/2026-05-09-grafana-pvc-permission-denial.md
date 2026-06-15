---
date: 2026-05-09
severity: medium
category: storage
affected: monitoring-grafana
status: resolved
---

# Grafana Init Container Permission Denial

## Summary

After migrating Grafana from local-path storage to Longhorn, the pod stuck in Init:CrashLoopBackOff. The init container chown failed with Permission denied on subdirectories. The volume contained pre-existing root-owned files from the previous local-path installation.

## Impact

- Grafana dashboard inaccessible for ~30 minutes
- No data loss (metrics are ephemeral; dashboards are in ConfigMaps)

## Timeline

- **10:00** Deleted old local-path PVC for Grafana
- **10:05** Helm recreated PVC with storageClassName: longhorn
- **10:10** Pod stuck Init:CrashLoopBackOff
- **10:15** Init container logs showed chown permission denied
- **10:25** Scaled Deployment to 0, deleted PVC, scaled back to 1
- **10:35** Pod started successfully with fresh volume

## Root Cause

Longhorn replicates filesystem metadata exactly. Previous local-path volume had root-owned files. Grafana init container runs as UID 472 and could not chown root-owned directories.

## Resolution

Accept data loss for Grafana local storage (dashboards are in ConfigMaps):

```bash
kubectl scale deployment monitoring-grafana -n monitoring --replicas=0
kubectl delete pvc monitoring-grafana -n monitoring
kubectl scale deployment monitoring-grafana -n monitoring --replicas=1
```

## Lessons Learned

1. Volume permissions are filesystem metadata and persist across StorageClass migrations
2. Inint containers are not root by default and cannot fix root-owned files. 
3. Grafana was safe to wipe because its state was in ConfigMaps. 

## Actions Taken
-[x] Document PVC recovery procedure in runbook
-[ ] Add init container securityContext review to Helm values audit
-[ ] f or future migrations, use kubectl cp or rsync to preserve permissions

## Supporting Evidence

Init container log:
```
	chown: changing ownership of '/var/lib/grafana/png': Permission denied
	chown: changing ownership of '/var/lib/grafana/csv': Permission denied
```

Deployment security context: 
```yaml
securityContext: 
  runAsUser: 472
  runAsGroup: 472
  fsGroupL 472
```

