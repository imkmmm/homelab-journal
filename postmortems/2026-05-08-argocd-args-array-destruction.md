---
date: 2026-05-08
severity: high
category: gitops
affected: argocd-server
status: resolved
---

# ArgoCD Server PR_END_OF_FILE_ERROR

## Summary

Applied `--insecure` to the ArgoCD server Deployment to simplify local
port-forward access. Used a `merge` patch strategy which replaced the
entire `args` array instead of appending to it. The server pod started
but immediately crashed on any connection, returning
`PR_END_OF_FILE_ERROR` to clients.

## Impact

- ArgoCD UI inaccessible via port-forward
- GitOps sync pipeline halted
- No ability to view or reconcile application state
- Duration: ~45 minutes

## Timeline (all times local)

- **14:00** Applied patch via `kubectl patch deployment argocd-server ... --type merge`
- **14:05** Port-forward to `localhost:8080` returned `PR_END_OF_FILE_ERROR`
- **14:10** Checked pod status: `Running` but logs showed immediate exits
- **14:20** Inspected live manifest: `args` array contained only `["--insecure"]`
- **14:30** Identified root cause: `merge` patch overwrote the array
- **14:35** Executed `kubectl rollout undo deployment/argocd-server -n argocd`
- **14:45** Re-applied using JSON Patch `op: add` with `path: .../args/-`
- **14:50** Verified with `kubectl get deployment ... -o jsonpath='{.spec.template.spec.containers[0].args}'`
- **15:00** Port-forward stable; UI accessible

## Root Cause

The `kubectl patch --type merge` command on a Deployment's `args` array
does **not** append; it replaces the entire field. ArgoCD server's
default args include internal service addresses (`--dex-server`,
`--repo-server`, `--redis`). Stripping these left the server unable to
communicate with its dependencies.

## Resolution

Rollback to previous revision, then apply correctly:

```bash
# Undo the destructive patch
kubectl rollout undo deployment/argocd-server -n argocd

kubectl patch deployment argocd-server -n argocd --type json \ -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--insecure"}]'

```
## Lessons Learned 

1. Arrays and merge `merge` patches are dangerous. Use `json` patch with `op:add`.

2. capture live manifests before mutating. A quick `kubectl get deployment <name> -o yaml > backup.yaml` before any patch.

3. Critical infrastrucure deserves a staging cluster. 

## Action Items

- [x] Document correct patch synatx in runbook 
- [ ] set up kind cluster for destructive GitOps experiments. 
- [ ] add pre-patch backups to personal workflow

##Supporting Evidence

Bad manifest (only element remaining):
```yaml
args:
- "--insecure"

```

Good manifest (appended correctly):

```yaml
args: 
- "/usr/local/bin/argocd-server"
- "--rootpath"
- "/argo-cd"
- "--insecure"
```





