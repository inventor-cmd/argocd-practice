# Incident Report: CoreDNS RBAC Failure Blocking ArgoCD Repository Registration

**Date:** March 29, 2026
**Severity:** High
**Status:** Resolved
**Environment:** k3s Kubernetes Cluster

---

## 1. Executive Summary

During the setup of ArgoCD on a k3s cluster, a repository could not be registered due to an internal DNS resolution failure. The root cause was CoreDNS being unable to authenticate with the Kubernetes API server, caused by a broken or missing RBAC ClusterRoleBinding. This prevented CoreDNS from listing services, endpoints, and namespaces, making all in-cluster DNS resolution fail. As a result, `argocd-repo-server` was unreachable by name, blocking ArgoCD from establishing connections to Git repositories.

---

## 2. Timeline of Events

| Time | Event |
|------|-------|
| T+0  | ArgoCD installed on k3s cluster using `kubectl apply` |
| T+1  | Attempt to register a Git repository in ArgoCD fails silently |
| T+2  | DNS test pod (`busybox:1.28`) run to probe `argocd-repo-server.argocd.svc.cluster.local` — resolution fails |
| T+3  | CoreDNS logs inspected — `Unauthorized` errors found against the Kubernetes API |
| T+4  | RBAC ClusterRole and ClusterRoleBinding for CoreDNS re-applied |
| T+5  | CoreDNS deployment restarted — logs clean, no more `Unauthorized` errors |
| T+6  | DNS resolution test re-run — `argocd-repo-server` resolves successfully |
| T+7  | ArgoCD repository registration succeeds — incident resolved |

---

## 3. Symptoms Observed

### 3.1 ArgoCD Repository Registration Failure

Attempting to add a repository via the ArgoCD CLI or UI failed. ArgoCD was unable to reach `argocd-repo-server` internally, which is the component responsible for cloning and indexing Git repositories.

### 3.2 DNS Resolution Failure

A DNS diagnostic pod confirmed the failure:

```
Server:    10.43.0.10
Address 1: 10.43.0.10 kube-dns.kube-system.svc.cluster.local

nslookup: can't resolve 'argocd-repo-server.argocd.svc.cluster.local'
```

Even though the `kube-dns` service itself was reachable, CoreDNS could not resolve any service names because it lacked the data to do so.

### 3.3 CoreDNS Logs — Unauthorized Errors

Inspection of CoreDNS pod logs revealed the root cause:

```
[ERROR] plugin/kubernetes: Failed to watch: failed to list *v1.Service: Unauthorized
[ERROR] plugin/kubernetes: Failed to watch: failed to list *v1.EndpointSlice: Unauthorized
[ERROR] plugin/kubernetes: Failed to watch: failed to list *v1.Namespace: Unauthorized
```

CoreDNS was continuously failing to watch Kubernetes resources, meaning its internal service cache was empty and no DNS records could be resolved.

---

## 4. Root Cause Analysis

### 4.1 Primary Cause

CoreDNS uses a Kubernetes ServiceAccount (`coredns` in the `kube-system` namespace) to watch the API server for services, endpoints, and namespaces. It requires a `ClusterRoleBinding` to grant it the necessary read permissions.

On this k3s cluster, the `ClusterRoleBinding` linking the `system:coredns` ClusterRole to the `coredns` ServiceAccount was either missing or had become invalid. Without this binding, every API watch request from CoreDNS was rejected with `401 Unauthorized`.

### 4.2 Why This Happens on k3s

k3s manages its own embedded CoreDNS deployment. The RBAC binding can be lost or broken in the following scenarios:

- A manual `kubectl apply` of CoreDNS or ArgoCD manifests that inadvertently overwrites cluster-scoped RBAC resources
- A k3s version upgrade that resets or changes internal RBAC configurations
- Applying the ArgoCD `install.yaml` manifest (which is large and contains many resources) may have triggered conflicts or partial overwrites of cluster-level RBAC objects
- The ServiceAccount token rotating or becoming stale without the binding being refreshed

### 4.3 Why It Blocked ArgoCD Specifically

ArgoCD's internal components communicate using Kubernetes DNS names (e.g., `argocd-repo-server.argocd.svc.cluster.local`). When CoreDNS has no service data, these names do not resolve, and the ArgoCD application controller and API server cannot reach `argocd-repo-server`. Repository registration is the first operation that exercises this communication path, making it the first visible symptom.

---

## 5. Impact

| Area | Impact |
|------|--------|
| ArgoCD repository registration | Completely blocked |
| In-cluster DNS resolution | Fully broken for all namespaces |
| Existing running workloads | Dependent on whether DNS was cached prior to failure |
| k3s cluster health | Cluster operational, control plane unaffected |

---

## 6. Resolution Steps

### Step 1 — Diagnose CoreDNS logs

```bash
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50
```

Confirmed `Unauthorized` errors against the Kubernetes API.

### Step 2 — Re-apply CoreDNS RBAC

The ClusterRole and ClusterRoleBinding were re-applied to restore API access:

```bash
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: system:coredns
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
rules:
- apiGroups: [""]
  resources: ["endpoints", "services", "pods", "namespaces"]
  verbs: ["list", "watch"]
- apiGroups: ["discovery.k8s.io"]
  resources: ["endpointslices"]
  verbs: ["list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:coredns
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:coredns
subjects:
- kind: ServiceAccount
  name: coredns
  namespace: kube-system
EOF
```

### Step 3 — Restart CoreDNS

```bash
kubectl rollout restart deployment coredns -n kube-system
kubectl get pods -n kube-system -l k8s-app=kube-dns -w
```

### Step 4 — Verify DNS Resolution

```bash
kubectl run dnstest --image=busybox:1.28 --rm -it --restart=Never -- \
  nslookup argocd-repo-server.argocd.svc.cluster.local
```

Resolution succeeded after the fix.

### Step 5 — Verify ArgoCD Repository Registration

Repository was successfully added to ArgoCD following DNS recovery.

---

## 7. Prevention & Recommendations

### 7.1 Use Server-Side Apply for Large Manifests

The initial ArgoCD installation used `kubectl apply`, which stores the full manifest in an annotation and can conflict with existing cluster-scoped resources. Use server-side apply to prevent this:

```bash
kubectl apply -n argocd \
  --server-side \
  --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

This also resolves the separate `262144 byte` annotation size error observed during CRD installation.

### 7.2 Add CoreDNS Health Check to Cluster Setup Checklist

After any cluster install or upgrade, verify CoreDNS can resolve services before proceeding:

```bash
kubectl run dnstest --image=busybox:1.28 --rm -it --restart=Never -- \
  nslookup kubernetes.default.svc.cluster.local
```

### 7.3 Monitor CoreDNS Logs Post-Install

Always inspect CoreDNS logs after installing large manifests like ArgoCD:

```bash
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=30
```

### 7.4 Pin the CoreDNS RBAC in GitOps

Store the CoreDNS ClusterRole and ClusterRoleBinding in your GitOps repository so it can be consistently applied and audited. This prevents silent drift.

### 7.5 Add `ServerSideApply=true` to ArgoCD SyncOptions

For all ArgoCD-managed applications, add this to prevent similar annotation-size issues on future resource syncs:

```yaml
syncPolicy:
  syncOptions:
    - ServerSideApply=true
```

---

## 8. Lessons Learned

- **DNS is foundational** — all ArgoCD inter-component communication depends on DNS. Any DNS failure will manifest as seemingly unrelated application errors, making diagnosis non-obvious.
- **k3s manages CoreDNS internally** — manual changes to cluster-scoped RBAC can silently break k3s-managed components.
- **`kubectl apply` is unsafe for large manifests** — server-side apply should be the default when working with CRDs and large operator manifests.
- **Log inspection is the fastest path to root cause** — the CoreDNS logs immediately and clearly identified the `Unauthorized` error, shortcutting what could have been a lengthy network debugging session.

---

## 9. References

- [ArgoCD Installation Docs](https://argo-cd.readthedocs.io/en/stable/getting_started/)
- [k3s CoreDNS Configuration](https://docs.k3s.io/networking/networking-services#coredns)
- [Kubernetes RBAC Documentation](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [CoreDNS Kubernetes Plugin](https://coredns.io/plugins/kubernetes/)

---

*Report generated on March 29, 2026*
