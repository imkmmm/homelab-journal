---
title: Fix PVC Permission Issues After StorageClass Migration
date: 2026-05-09
---

# Fix PVC Permission Issues After StorageClass Migration

## Symptoms

- Pod stuck in `Init:CrashLoopBackOff`
- Init container logs show `Permission denied` on `chown` operations
- Volume migrated from one StorageClass to another (e.g., `local-path` → `longhorn`)

## Root Cause

Filesystem permissions from the old volume persist onto the new volume.
If the old volume had `root`-owned files, the new init container (running
as non-root) cannot `chown` them.

## Recovery Steps

### If Data Loss is Acceptable

```bash
# Scale to zero to release the volume
kubectl scale deployment &lt;name&gt; -n &lt;namespace&gt; --replicas=0

# Delete the PVC
kubectl delete pvc &lt;pvc-name&gt; -n &lt;namespace&gt;

# Let Helm recreate fresh
kubectl scale deployment &lt;name&gt; -n &lt;namespace&gt; --replicas=1
```


### If Data Must Be Presevered

```bash
# Run a debug pod with root access 
kubectl run debug --rm -it --image=busybox --overrides='{
  "spec": {
    "containers": [{
      "name": "debug",
      "image": "busybox",
      "command": ["sh"],
      "volumeMounts": [{"name": "data", "mountPath": "/data"}],
      "securityContext": {"runAsUser": 0}
    }],
    "volumes": [{"name": "data", "persistentVolumeClaim": {"claimName": "<pvc-name>"}}]
  }
}' -- /bin/sh

# Inside the debug pod, fix permissions
chown -R <target-uid>:<target-gid> /data
exit
```

## Prevention
- Always check 'securityContext' and 'fsGroup'before migrating storage
- Use 'kubectl cp' or 'rsync -a' to preserve permissions during migration
- Test init container behavior on a copy of the data before cutting over
