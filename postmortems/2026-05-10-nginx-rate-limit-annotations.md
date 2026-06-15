date: 2026-05-10
severity: low
category: security
affected: public ingresses (hello-world, homarr, uptime-kuma)
status: resolved
---

# NGINX Rate Limit Annotation Confusion

## Summary

Mixed commercial NGINX Inc annotations with community ingress-nginx annotations. Commercial annotations were silently ignored. Using limit-rps and limit-rpm together caused the controller to skip applying limit_req entirely. A typo in one Ingress also caused that resource to be skipped.

## Impact

- No rate limiting enforced for ~2 days
- False sense of security
- No service degradation (silent failure)

## Timeline

- **Day 1** Applied limit-req-rate and limit-req-burst to Ingresses
- **Day 2** Load testing showed no 503 responses
- **Day 3** Reviewed controller logs; no limit_req zones created
- **Day 3** Removed commercial annotations; fixed typo in uptime-kuma Ingress
- **Day 3** Verified with internal ClusterIP flood test

## Root Cause

Did not verify which Ingress controller flavor was installed. The cluster uses community kubernetes/ingress-nginx, not nginxinc/kubernetes-ingress. These projects share similar names but divergent annotation sets. Additionally, limit-rps and limit-rpm conflict in the community controller.

## Resolution

Stripped all Ingresses to validated community annotations:

| Service | limit-rps | limit-connections |
|---------|-----------|-------------------|
| hello-world | 20 | 50 |
| homarr | 20 | 50 |
| uptime-kuma | 20 | 50 |
| grafana | 5 | 10 |

Testing via internal ClusterIP:

```bash
kubectl run -it --rm flood --image=curlimages/curl --restart=Never -- \
  sh -c 'seq 1 150 | xargs -P50 -I{} curl -s -o /dev/null -w "%{http_code}\n" \
    -H "Host: hello.mydomain.tld" \
    http://ingress-nginx-controller.ingress-nginx.svc.cluster.local/'
```
##Lessons Learned 
1. Silent ignore is the worst failure. Always verify with test
2. Annoatation namespaces matter. nginx.ingress.kubernetes.kubernetes.io/ vs nginx.org/
3. Test from inside the cluster. Cloudflare Tunnel does not hairpin NAT

##Actions Items
-[x] Audit all Ingresses for commercial annotation contamination
-[ ] Create crashlab ingress for rate-limit testing


##Supporting Evidence

Invalid annotations removed: 

```yaml
annotations:
  nginx.ingress.kubernetes.io/limit-req-rate: "100"      # INVALID
  nginx.ingress.kubernetes.io/limit-req-burst: "50"       # INVALID
  nginx.ingress.kubernetes.io/limit-rps: "20"            # VALID
  nginx.ingress.kubernetes.io/limit-rpm: "1200"           # CONFLICTS
  nginx.ingress.kubernetes.io/limit-conncetions: "50"     # TYPO (lol)
```
Valid annotaions applied:

```yaml 
annotations:
  nginx.ingress.kubernetes.io/limit-rps: "20"
  nginx.ingress.kubernetes.io/limit-burst-multiplier: "5"
  nginx.ingress.kubernetes.io/limit-connections: "50"
```

## Technical Context & Future Direction

> **⚠️ Critical Notice:** The community Kubernetes NGINX Ingress Controller (`kubernetes/ingress-nginx`) was officially retired in March 2026. It has reached End-of-Life (EOL) and will no longer receive updates, bug fixes, or security patches [[10]]. Continuing to use it for internet-facing workloads is a significant security risk.

**Recommended Next Steps:**
- **Immediate:** Plan a migration to [Gateway API](https://gateway-api.sigs.k8s.io/), the official Kubernetes-native successor to the Ingress resource.
- **Alternative Controllers:** Evaluate actively maintained Gateway API implementations like Cilium Gateway, HAProxy Ingress, Traefik, or Envoy Gateway.
