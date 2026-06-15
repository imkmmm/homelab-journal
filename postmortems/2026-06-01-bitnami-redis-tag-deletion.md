---
date: 2026-05-12
severity: medium
category: supply-chain
affected: immich-redis
status: resolved
---

# Bitnami Redis Image Tag Deletion

## Summary

The Immich chart originally specified a Bitnami Redis subchart. The Redis pod entered ImagePullBackOff because the exact Docker image tag had been deleted from Docker Hub by Bitnami.

## Impact

- Immich cache layer unavailable
- Immich server could not start
- Duration: ~15 minutes

## Timeline

- **11:00** Deployed Immich via ArgoCD
- **11:05** Redis pod status: ImagePullBackOff
- **11:10** kubectl describe showed not found for Bitnami tag
- **11:15** Identified Bitnami's aggressive tag retention policy
- **11:20** Switched to official redis:7.2-alpine image
- **11:25** Redis pod running; Immich server started

## Root Cause

**Primary Cause:** Bitnami deletes old Docker image tags frequently to save registry space. A Helm chart pinned to a specific Bitnami tag will eventually fail to pull even if the chart is unchanged.

**Contributing Factor:** The initial deployment used Bitnami Redis because AI-generated tutorials and documentation commonly recommend Bitnami Helm charts as the "standard" approach for Kubernetes deployments. Bitnami charts are widely popular, well-documented, and convenient for gett… convenience came with hidden risks:
- Bitnami's aggressive tag retention policy is not prominently documented
- AI tutorials rarely mention the ephemeral nature of Bitnami tags
- The decision prioritized ease of deployment over long-term stability
- No critical evaluation was performed on whether Bitnami's tag management strategy aligned with production requirements

This reflects a broader pattern which is following popular recommendations without validating their assumptions for production use cases.

## Resolution

Disabled the Bitnami Redis subchart and deployed Redis manually using the official Docker Library image:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: immich-redis-master
  namespace: immich
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7.2-alpine
        ports:
        - containerPort: 6379
```
Updated Immich chart values to point to manual Redis service:

```yaml
REDIS_HOSTNAME: immich-redis-master
```
##Lessons Learned
1. Bitnami tags are ephemeral. Not reliable for production
2. Official library images are safer. Tags are not deleted
3. Subcharts are convenient until they break. 

##Actions Item
- [x] Replace all Bitnami images with official or self-mirrored images
- [ ] Set up container image mirror to insulate from upstream tag deletion
- [ ] Add image pull failure alerting to Prometheus

##Supporting Evidence 
Pod event obtained from event logs in pods in argocd ui:

```
Failed to pull image "docker.io/bitnami/redis:7.2.5-debian-12-r0":
rpc error: code = NotFound desc = failed to pull and unpack image:
docker.io/bitnami/redis:7.2.5-debian-12-r0: not found
```
