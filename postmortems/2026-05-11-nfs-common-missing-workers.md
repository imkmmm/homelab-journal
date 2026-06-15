---
date: 2026-05-11
severity: high
category: storage
affected: nfs-provisioner, immich-library pvc
status: resolved
---

# NFS-Common Missing on Kubernetes Worker Nodes

## Summary

After installing the NFS subdir external provisioner and creating the immich-library PVC, the provisioner pod and any pod attempting to mount the PVC got stuck in ContainerCreating. Kubelet logged mount failed: exit status 32 because /sbin/mount.nfs did not exist on worker nodes.

## Impact

- Immich deployment blocked for ~20 minutes
- NFS-backed PVCs could not be scheduled on any worker

## Timeline

- **16:00** Installed NFS provisioner Helm chart
- **16:05** Created immich-library PVC; provisioner pod stuck
- **16:10** kubectl describe pod showed MountVolume.SetUp failed
- **16:15** Checked worker: which mount.nfs returned nothing
- **16:20** Installed nfs-common on all workers
- **16:25** PVC bound successfully

## Root Cause

Ubuntu cloud images do not include nfs-common by default. Kubernetes does not install OS-level NFS client tools. The kubelet calls the host's mount binary, which delegates to /sbin/mount.nfs for NFS filesystems.

## Resolution

Install NFS client on every node that might schedule NFS-backed pods:

```bash
sudo apt update && sudo apt install -y nfs-common
```
##Lessons Learned

1. StorageClass readiness does not equal node readiness
2. Day-0 infrastructure must be documented; this should be in the node preparation runbook
3. NFS relies on host OS kernel module and userspace utilities.

##Actions Items 

-[x]  Add nfs-common installation to VM preparation script
-[ ] Create node-readiness DaemonSet that verifiees storage helpers

##Supporting Evidence

Kubelet event on stuck pod (shown in ArgoCD immich namespace on pod event logs):

```
Warning  FailedMount  12s  kubelet  MountVolume.SetUp failed for volume "pvc-xxx-yyy":
mount failed: exit status 32
Output: mount: /var/lib/kubelet/pods/...: bad option; for several filesystems
(e.g. nfs, cifs) you might need a /sbin/mount.<type> helper program.
```
