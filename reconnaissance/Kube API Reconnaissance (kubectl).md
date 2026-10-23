
# Basics

List API resources (e.g., CRDs) available, including their shortcuts
```bash
kubectl api-resources
kubectl cluster-info
kubectl get namespaces
```

Output format can be specified with:
```bash
kubectl get pods -o wide # more details in the table
kubectl get pods -o [yaml|json]
```

``kubernetes.default.svc.cluster.local`` is the standard kube API DNS name, which is usually stored in a ``KUBERNETES_SERVICE_HOST`` env var in pods.
# Identity and Permissions

```bash
kubectl auth whoami
kubectl auth whoami --token=$TOKEN

# Check if you can 'get' pods in the default namespace
kubectl auth can-i get pod

# List permissions (namespace-scoped)
`kubectl auth can-i --list -n kube-system`

# Check permissions for another user
kubectl auth can-i get pod —as system:serviceaccount:secure:unprivileged-service-account

# View namespace-scoped roles
kubectl get roles
kubectl describe role dev

# View cluster-scoped role bindings
kubectl get rolebindings
kubectl describe rolebinding dev-binding
```

Get cluster role bindings for a specific service account:
```bash
kubectl get rolebindings,clusterrolebindings -A -o custom-columns='KIND:kind,NAMESPACE:metadata.namespace,NAME:metadata.name,SERVICE_ACCOUNTS:subjects[?(@.kind=="ServiceAccount")].name' | grep "<SERVICE_ACCOUNT_NAME>"
```

## What can my token do?

List your token's permissions (above) and review [[Kubernetes RBAC]] to see what is possible

# Pods
```bash
# List pods in default namespace
kubectl get pods

# List pods in specific namespace
kubectl get pods -n <namespace>

# List pods in all namespaces
kubectl get pods -A

# Find pod by IP
`kubectl get --all-namespaces --output json pods | jq '.items[] | select(.status.podIP=="10.22.19.69")' | jq .metadata.name`
```

## List Containers Running in a Pod

```bash
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.containers[*].name}'
```

## List Pods and Their Service Accounts:

```bash
# List all pods with their service accounts
kubectl get pods -A -o custom-columns=NAMESPACE:.metadata.namespace,POD:.metadata.name,SERVICE_ACCOUNT:.spec.serviceAccountName

# And pod uid by node
kubectl get pods -A --token=$TOKEN --field-selector spec.nodeName=<NODE-NAME> -o custom-columns=NAMESPACE:.metadata.namespace,POD:.metadata.name,SERVICE_ACCOUNT:.spec.serviceAccountName,UID:.metadata.uid

# Same but sorted by namespace
kubectl get pods -A -o custom-columns=NAMESPACE:.metadata.namespace,POD:.metadata.name,SERVICE_ACCOUNT:.spec.serviceAccountName --sort-by=.metadata.namespace

# Include node information
kubectl get pods -A -o custom-columns=NAMESPACE:.metadata.namespace,POD:.metadata.name,SERVICE_ACCOUNT:.spec.serviceAccountName,NODE:.spec.nodeName --sort-by=.metadata.namespace
```

## Secrets

Secrets can be dumped individually or en masse by listing them with `-o yaml` or `-o json` 
```bash
# Dump all creds in default namespace
kubectl get secrets -o yaml

# Dump creds for all namespaces
kubectl get secrets -o yaml -A
```



# Nodes

```bash
kubectl get nodes

# Find control plane nodes
kubectl get node --selector='node-role.kubernetes.io/control-plane'
```

## Namespace

```
kubectl get ns
```

See [[Things of interest]]

# Custom Resource Definitions (CRD)
Various cluster integrations or operators create CRDs, which are user-set types like `pod` or `node` that can be managed with `kubectl`. They may be used to stored secrets that could be beneficial to your attack path.

CRDs are monitored by an **operator**, which takes actions based on that CRD's resources. Practically, an operator is just a pod with access to the CRD and the ability to take the actions it needs to in the cluster or on external resources. 

You can list CRDs by running:
```bash
kubectl api-resources
```

Here is an example custom resource definition:
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: traintickets.trains.com
spec:
  group: trains.com
  scope: Namespaced
  names:
    plural: traintickets
    singular: trainticket
    kind: TrainTicket
    shortNames:
      - tt
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                from:
                  type: string
                to:
                  type: string
                number:
                  type: integer
                  minimum: 1
```

And this is an instance of that CRD
```yaml
apiVersion: trains.com/v1
kind: TrainTicket
metadata:
  name: my-train-ticket
spec:
  from: Tokyo
  to: Kyoto
  number: 2
```

Then you can query them like any other resource:
```
kubectl get flighttickets -o yaml
```