---
date: 2026-06-02
severity: medium
category: networking
affected: immich-server ingress
status: resolved
---

# Ingress Missing ingressClassName

## Summary

After deploying Immich via ArgoCD, browsing to photos.mydomain.tld returned 404 Not Found. The Ingress resource showed CLASS: none instead of nginx. The Immich Helm chart did not correctly apply the ingressClassName field.

## Impact

- Immich web UI and mobile app inaccessible
- False impression that Cloudflare Tunnel or DNS was misconfigured
- Duration: ~25 minutes

## Timeline

- **13:00** ArgoCD synced Immich application
- **13:05** DNS resolved but returned 404
- **13:10** Cloudflare Tunnel logs healthy
- **13:15** kubectl get ingress showed CLASS: none
- **13:20** Identified Helm chart bug
- **13:25** Applied manual JSON patch
- **13:30** 404 resolved

## Root Cause

The Immich Helm chart v0.12.0 has a templating bug where ingressClassName in values.yaml is not correctly rendered into the Ingress manifest. Without this field, the NGINX Ingress Controller ignores the resource entirely.

## Resolution

Manual patch after Helm deployment:

```bash
kubectl patch ingress immich-server -n immich \
  --type='json' \
  -p='[{"op": "add", "path": "/spec/ingressClassName", "value":"nginx"}]'
```

## Lessons Learned
1. Helm charts have bugs. Always helmp template and inspect output. 
2. ingressClassName is mandatory
3. 404 from the cluster is different from 404 from Cloudflare

## Actions Items
[x] Document manual patch in runbook

## Supporting evidence 

Ingress before patch:
```yaml 
spec:
  rules:
  - host: photos.mydomain.tld
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: immich-server
            port:
              number: 2283
  # ingressClassName is MISSING
```
NGINX controller log:
```
Ignoring ingress immich/immich-server because it does not contain a valid IngressClass
```
