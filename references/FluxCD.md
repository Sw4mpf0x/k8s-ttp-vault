### Typical Default FluxCD Deployment

A standard Flux installation creates one namespace where all Flux controllers live here by default:

```
flux-system
```

### Core CRDs Involved

**GitRepository** — defines what to fetch:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: my-app
  namespace: flux-system
spec:
  interval: 1m
  url: https://github.com/org/repo
  ref:
    branch: main        # or tag, semver, commit
  secretRef:
    name: github-creds  # SSH key or PAT
```

**Kustomization** — defines what to apply:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: my-app
  namespace: flux-system
spec:
  interval: 5m
  sourceRef:
    kind: GitRepository
    name: my-app
  path: ./clusters/production
  prune: true           # delete resources removed from Git
  targetNamespace: my-app
```

# Finding FluxCD Configuration in a Cluster

NOTE: The `flux` CLI uses the same kubeconfig as `kubectl` by default.
## Quick Overview of All Flux Resources

```bash
# See all Flux CRDs installed
kubectl get crds | grep flux

# Get all Flux resources across all namespaces
flux get all -A

# Or with kubectl
kubectl get all -n flux-system
```

---

## Core Resource Types to Inspect

### 1. **GitRepositories / OCIRepositories / HelmRepositories** (Source Config)
```bash
# List all sources
flux get sources all -A

# Get details
kubectl get gitrepository -A
kubectl describe gitrepository flux-system -n flux-system

# See the URL, branch, secret references
kubectl get gitrepository -n flux-system -o yaml
```

### 2. **Kustomizations** (What gets deployed & from where)
```bash
flux get kustomizations -A

kubectl get kustomization -A -o yaml
# Shows:
#   - sourceRef (which repo)
#   - path (folder in repo)
#   - interval
#   - targetNamespace
```

### 3. **HelmReleases** (Helm-based deployments)
```bash
flux get helmreleases -A

kubectl get helmrelease -A -o yaml
# Shows:
#   - chart source
#   - values
#   - targetNamespace
```

---

## Finding Credentials Configuration

```bash
# Find what secrets Flux is referencing
kubectl get gitrepository -A -o jsonpath='{range .items[*]}{.metadata.namespace}{"\t"}{.metadata.name}{"\t"}{.spec.secretRef.name}{"\n"}{end}'

# Check service accounts used
kubectl get kustomization -A -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.serviceAccountName}{"\n"}{end}'
```

---

## Flux System Configuration

```bash
# Check Flux controllers and their config
kubectl get deployment -n flux-system -o yaml

# Check Flux version and components
flux version
flux check

# ConfigMaps
kubectl get configmap -n flux-system -o yaml

# Check Flux feature flags / args
kubectl get deployment source-controller -n flux-system -o jsonpath='{.spec.template.spec.containers[0].args}'
```

---

## Reconciliation & Sync Status

```bash
# See sync status of everything
flux get all -A

# See events (errors, successes)
kubectl get events -n flux-system --sort-by='.lastTimestamp'

# Flux logs
kubectl logs -n flux-system deploy/source-controller
kubectl logs -n flux-system deploy/kustomize-controller
kubectl logs -n flux-system deploy/helm-controller
kubectl logs -n flux-system deploy/notification-controller
```

---

## Notifications & Alerts Config

```bash
# Providers (Slack, Teams, webhooks, etc.)
kubectl get provider -A -o yaml

# Alert configurations
kubectl get alert -A -o yaml

# Receivers (inbound webhooks)
kubectl get receiver -A -o yaml
```

---

## Image Automation (if enabled)

```bash
kubectl get imagerepository -A
kubectl get imagepolicy -A
kubectl get imageupdateautomation -A -o yaml
# Shows which repo/branch Flux is writing image updates to
```

---

## Useful One-Liners

```bash
# Full dump of all Flux resources
kubectl get $(kubectl api-resources | grep toolkit.fluxcd.io | awk '{print $1}' | tr '\n' ',') -A -o yaml 2>/dev/null

# Find the Git repo URL and branch Flux is syncing from
kubectl get gitrepository -A -o jsonpath='{range .items[*]}{.metadata.name}{": "}{.spec.url}{" ("}{.spec.ref}{")"}{"\n"}{end}'

# Find all referenced secrets
kubectl get gitrepository,helmrepository,ocirepository -A -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.secretRef.name}{"\n"}{end}'
```
