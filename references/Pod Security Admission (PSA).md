* The PSA is a built-in pod admission controller (introduced in v1.25) that enforces Pod Security Standards (PSS) to control the types of pods that can be deployed to a namespace.
* PSS are namespace-scoped and controlled via namespace labels
* When a pod is created, PSA checks its spec against the namespace’s configured security level and either:
	* Allows
	* Warns
	* Rejects
* There are 3 baseline profiles
	* Privileged – No restrictions
	* Baseline – Blocks the most egregious escapes (hostPID, hostNetwork, privileged containers, most host volume types)
	* Restricted – Strictest settings; requires non-root, drops all capabilities, enforces seccomp, no privilege escalation
* There are 3 modes that control how a pod is handled on violation
	* enforce – Reject the pod
	* audit – Allow, but log violation
	* warn – Allow, but warn the client

## View Namespace PSS
Check PSA by querying the namespace:
```bash
kubectl get ns <namespace> -o yaml
```
## PSS Examples:
Fully locked down, reject anything not restricted
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
```

Warn + audit on restricted
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: migrating
  labels:
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: latest
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: latest
```

No restrictions
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: kube-system
  labels:
    pod-security.kubernetes.io/enforce: privileged
```