# Built-in Roles
### Service Account Format

```
system:serviceaccount:namespace:name
# Example: system:serviceaccount:kube-system:default
```
### System Roles (prefix: `system:`)

| Role                                  | Description                                                                                                                                                                                                                                             |
| ------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| system:masters (group)                | Allows super-user access to perform any action on any resource.                                                                                                                                                                                         |
| system:authenticated (group)          | Allows a user read-only access to basic information about themselves.                                                                                                                                                                                   |
| system:kube-scheduler (user)          | Allows access to the resources required by the [scheduler](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-scheduler/) component.                                                                                                |
| system:kube-scheduler (user)          | Allows access to the volume resources required by the kube-scheduler component.                                                                                                                                                                         |
| system:kube-controller-manager (user) | Allows access to the resources required by the [controller manager](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-controller-manager/) component.                                                                              |
| system:node:<node-name>               | Allows access to resources required by the kubelet, including read access to all secrets, and write access to all pod status objects. The system:node role only exists for compatibility with Kubernetes clusters upgraded from versions prior to v1.8. |
| system:kube-proxy (user)              | Allows access to the resources required by the [kube](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/)[-proxy](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/) component.                  |

# Permissions

| Resource (verb)                                                                                   | Capability it unlocks                                                                                                                                                                                                                                                                                                    |
| ------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `secrets` (get/list/watch)                                                                        | [[Kube API Reconnaissance (kubectl)#Secrets\|Harvest all secrets in scope]]                                                                                                                                                                                                                                              |
| `pods` (create)                                                                                   | [[Pod Management\|Schedule arbitrary pods]]; potential escalation to node access, access to namespace secrets                                                                                                                                                                                                            |
| `pods/exec`<br>`pods/attach` (create)                                                             | [[Pod Management#Execute Commands in Target Pod\|Execute Commands in Target Pod]]                                                                                                                                                                                                                                        |
| `pods/ephemeralcontainers` (update/patch)                                                         | [[Pod Management#Create Ephemeral Container in Target Pod\|Inject a debug container into a running pod]]<br>Access target service accounts, potentially escalate to node access via privileged container                                                                                                                 |
| `pods/portforward` (create)                                                                       | [[Pod Management#Port Forward\|Tunnel to pod-local services]], bypassing NetworkPolicy because it goes through the kube API                                                                                                                                                                                              |
| `serviceaccounts/token` (create)                                                                  | [[Service Accounts#Create Service Account Token\|Mint tokens for any service account you can name]]                                                                                                                                                                                                                      |
| `serviceaccounts` (create/update)                                                                 | [[Service Accounts#Create Service Account\|Create a privileged service account]], escalate account in the cluster                                                                                                                                                                                                        |
| `*/*` wildcard                                                                                    | Effectively namespace admin, do anything                                                                                                                                                                                                                                                                                 |
| `clusterrolebindings` <br>`rolebindings` (create)                                                 | [[Service Accounts#3. Create Role Binding\|Bind your service account to a privileged role]] ***IF*** you also have the `bind` verb allowing it. By default, you cannot create a RoleBinding/ClusterRoleBinding that references a role granting permissions you don’t already hold. See [[#Bind Example\|example]] below. |
| `clusterroles` <br>`roles` (create/update)                                                        | [[Service Accounts#2. Create or Identify Role\|Create a privileged role]] ***IF*** you also have the `escalate` verb allowing it. By default, you cannot create a Role/ClusterRole containing permissions you don’t already hold. See [[#Escalate Requirement\|example]] below.                                          |
| `nodes/proxy` (get/create)                                                                        | [[Pod Management#Exploiting nodes/proxy\|Hit the kubelet API directly to exec into any pod on that node]], bypassing API-server RBAC and audit logging                                                                                                                                                                   |
| `nodes` (get/list/patch)                                                                          | Node recon (i.e., `kubectl get ns`); patching labels/taints enables scheduling manipulation                                                                                                                                                                                                                              |
| `certificatesigningrequests` (create) +<br>`/approval` (update) + `sign`                          | [[Certificates\|Issue a client cert for an arbitrary CN/Org]] — `O=system:masters` = instant cluster-admin                                                                                                                                                                                                               |
| `validatingwebhookconfigurations` <br>`mutatingwebhookconfigurations` (create)                    | Register a webhook that MITMs all API traffic — exfil secrets, silently backdoor objects                                                                                                                                                                                                                                 |
| `apiservices` (create)                                                                            | Aggregation-layer API registration → intercept API groups                                                                                                                                                                                                                                                                |
| `deployments`<br>`daemonsets`<br>`statefulsets`<br>`replicasets`<br>`jobs`<br>`cronjobs` (create) | **Indirect pod creation** — controllers spawn pods, so these grant everything “create pods” does and often slip past policies that only restrict `pods`                                                                                                                                                                  |
| `configmaps` (get/list)                                                                           | Config recon; can contain credentials people shouldn’t have put there                                                                                                                                                                                                                                                    |
| `services` <br>`endpoints`<br>`endpointslices` (create/update)                                    | Traffic hijacking — repoint a Service’s endpoints at a pod you control                                                                                                                                                                                                                                                   |
| `persistentvolumes` (create)                                                                      | Define `hostPath` PVs → node filesystem access                                                                                                                                                                                                                                                                           |
| `networkpolicies` (create/update/delete)                                                          | Tear down or rewrite segmentation                                                                                                                                                                                                                                                                                        |
| `leases` (coordination.k8s.io, update/delete)                                                     | Disrupt controller leader election (DoS)                                                                                                                                                                                                                                                                                 |
| `events` (list)                                                                                   | Recon — reveals what’s running, image names, failures                                                                                                                                                                                                                                                                    |
| `namespaces` (list)                                                                               | Recon of cluster layout                                                                                                                                                                                                                                                                                                  |

## Bind Requirement
 By default, you cannot create a `RoleBinding`/`ClusterRoleBinding` that references a role granting permissions you don’t already hold.
 
 `bind` lets you create a `RoleBinding`/`ClusterRoleBinding` referencing a role whose permissions exceed your own. Below is an example of a role that can create bindings, but only to `cluster-admin`. If `resourceNames` is absent, you can bind to any role.
 ```yaml
 apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: binding-allowed
rules:
- apiGroups: ["rbac.authorization.k8s.io"]
  resources: ["clusterrolebindings"]
  verbs: ["create"]
- apiGroups: ["rbac.authorization.k8s.io"]
  resources: ["clusterroles"]
  verbs: ["bind"]
  resourceNames: ["cluster-admin"] 
 ```

## Escalate Requirement
By default, you cannot create a `Role`/`ClusterRole` containing permissions you don’t already hold.  `escalate` lets you create a `Role`/`ClusterRole`  whose permissions exceed your own. 

Below is an example of a role that can create roles with permissions above their own:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: role-manager
rules:
- apiGroups: ["rbac.authorization.k8s.io"]
  resources: ["clusterroles", "roles"]
  verbs: ["get", "list", "create", "update", "patch", "escalate"]
- apiGroups: ["rbac.authorization.k8s.io"]
  resources: ["clusterrolebindings", "rolebindings"]
  verbs: ["get", "list", "create"]
```

`escalate` also lets you expand a role that already exists, which is stealthier than creating new roles/bindings — modifying an existing legitimate role leaves fewer new objects in audit trails.:
```yaml
kubectl patch clusterrole <existing-role> --type=json -p='[
  {
    "op": "add",
    "path": "/rules/-",
    "value": {
      "apiGroups": [""],
      "resources": ["secrets"],
      "verbs": ["get", "list", "watch"]
    }
  }
]'
```