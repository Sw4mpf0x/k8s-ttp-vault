* Each namespace has a `default` service account that is automatically mounted to pods deployed there
* The default service account, created for each namespace can access only a few low-risk kube API endpoints, like `/version` or `/healthz`, by default.
* Service account tokens are provided to pods via a volume that is mounted to the pod at:
	* `/var/run/secrets/kubernetes.io/serviceaccount/token`
* Set a different service account for a pod with:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-kubernetes-dashboard
spec:
  serviceAccountName: dashboard-sa
  containers:
    - name: my-kubernetes-dashboard
      image: my-kubernetes-dashboard
```
# Create Service Account Token
Modern Kubernetes automatically rotates mounted service account tokens. 

Check [[Things of interest#Service Accounts|service accounts of interest]] if trying to escalate

 To create a persistent token by manifest:
```yaml
apiVersion: v1
kind: Secret
type kubernetes.io/service-account-token
metadata:
  name: mysecretname
  annotations:
    kubernetes.io/service-account.name: dashboard-sa
```

To create tokens with `kubectl`:
```bash
# List all SAs in namespace
kubectl get sa -n <ns>

# Find what each is bound to
kubectl get rolebindings,clusterrolebindings -A -o json | \
  jq '.items[] | select(.subjects[]?.name == "<sa-name>") | 
      {name: .metadata.name, role: .roleRef.name}'

# Create and print token
kubectl create token <privileged-sa> -n <namespace> --duration=8760h | jq -r '.status.token'

# Create and set token to env var
TOKEN=$(kubectl create token <privileged-sa> -n <namespace> --duration=8760h)

# Bound to a specific audience (for IRSA, Workload Identity, etc.)
kubectl create token <sa-name> -n <ns> --audience=https://kubernetes.default.svc -o json | jq -r '.status.token'
```

# Create Service Account

This is done in three steps:
1. Create Service Account
2. Create or identify a target role
3. Bind the service account to the role

## 1. Create Service Account

```bash
kubectl create serviceaccount my-sa -n <namespace>
```
## 2. Create or Identify Role
Roles are set within a given namespace. As such, you can’t apply then to cluster-scoped resources like nodes and persistent volumes.

See [[Kubernetes RBAC]] for more information on permissions and built-in roles

To enumerate roles, see [[Kube API Reconnaissance (kubectl)#Identity and Permissions]]

Example Role manifest that can create pods:
```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: developer
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["create", "delete", "list"]
```
## 3. Create Role Binding
Via CLI:
```bash
kubectl create rolebinding jupyter-admin-binding --clusterrole=admin --user=jupyter --namespace=default

kubectl create rolebinding jupyter-svc-admin-binding --clusterrole=admin --serviceaccount=<namespace>:my-sa --namespace=default
```

Via manifest:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: devuser-developer-binding
subjects:
- kind: User
  name: dev-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

## 4. Mint a Token
[[Service Accounts#Create Service Account Token| Mint a token for the service account]]
# ClusterRole
Roles for cluster resources and assigning access to namespace-scoped resources across all namespaces.

List namespace-scoped and non-namespace-scoped resources:
```bash
kubectl api-resources --namespaced=true
kubectl api-resources --namespaced=false
```

## Create ClusterRole
Role manifest:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-administrator
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["list", "get", "create", "delete"]
```

Binding manifest:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: node-admin-role-binding
subjects:
- kind: User
  name: michelle
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: node-administrator
  apiGroup: rbac.authorization.k8s.io
```

