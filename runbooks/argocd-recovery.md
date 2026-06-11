---
title: Recover ArgoCD After Bad Deployment Patch
date: 2026-05-08
---

# Recover ArgoCD After Bad Deployment Patch

## Symptoms

- `kubectl port-forward svc/argocd-server -n argocd 8080:80` connects but returns `PR_END_OF_FILE_ERROR`
- ArgoCD UI inaccessible
- Pod status shows `Running` but logs show immediate exits

## Root Cause

A bad `kubectl patch` replaced the `args` array, stripping internal
service addresses (`--dex-server`, `--repo-server`, `--redis`).

## Recovery Steps

### Step 1: Rollback

```bash
kubectl rollout undo deployment/argocd-server -n argocd
```
### Step 2: Wait for Ready

```bash
kubectl wait --for=condition=ready pod \
  -l app.kubernetes.io/name=argocd-server -n argocd \
  --timeout=120s
  ```

### Step 3: Apply '--insecure' Correctly 

```bash
kubectl patch deployment argocd-server -n argocd --type json \
  -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--insecure"}]'
```

### Step 4: Verify

```bash 
kubectl get deployment argocd-server -n argocd \
  -o jsonpath='{.spec.template.spec.containers[0].args}'
```

Expected: '["/usr/local/bin/argocd-server", ..., "--insecure"]'

## Prevention
- Always use 'json' patch with 'op:add' for arrays
- Capture backup before patching 'kubectl get deployment ... -o yaml > backup.yaml'
- test patches on a 'kind' cluster first
