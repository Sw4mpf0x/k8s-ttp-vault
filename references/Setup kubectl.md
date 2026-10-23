# Download kubectl
***
## To Pod/Node

Download `kubectl`:
```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
```

## To Assessment VM
Download 
```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
```

Download the k8s service account token and CA certificate to the assessment VM
```bash
# Point to local SOCKS proxy port
export HTTPS_PROXY=socks5://127.0.0.1:7005
export HTTP_PROXY=socks5://127.0.0.1:7005
export ALL_PROXY=socks5://127.0.0.1:7005

# Specify kube API host and port, otherwise it tries localhost
export KUBERNETES_SERVICE_HOST="kubernetes.default.svc.cluster.local"
export KUBERNETES_SERVICE_PORT="443"

kubectl --token=$(cat token) \
  --certificate-authority=ca.crt \
  --server=https://kubernetes.default.svc.cluster.local \
```

# Setup kubeconfig
***
`kubectl` uses a kubeconfig to determine cluster and user information. It is composed of 3 sections:

Clusters
- Info on clusters like server address
- Can list namespaces
Contexts
- ties clusters and users together
Users
- Authentication information for each user such as certs (maybe?) and cert locations, sso info, or tokens

Default kubeconfig locations:
* `$HOME/.kube/config`
* On a node:
	* /etc/kubernetes/kubelet.conf
	* /var/lib/kubelet/kubeconfig

View current config: `kubectl config view` 

### Option 1: CLI kubeconfig Arg
Set a custom config file with: 
```
--kubeconfig=/path/to/kubeconfig
```

### Option 2: Env var
Or use an env var prior to using `kubectl`:
```bash
export KUBECONFIG=/path/to/kubeconfig
```

### Option 3: Setup kubeconfig Context

```bash
# Step 1: Set cluster
kubectl config set-cluster thiscluster \
  --server=https://kubernetes.default.svc:443 \
  --certificate-authority=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt

# Step 2: Set credentials (using token)
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
kubectl config set-credentials thisnode \
  --token=$TOKEN

# Step 3: Set context
kubectl config set-context my-context \
  --cluster=thiscluster \
  --user=thisnode \
  --namespace=default

# Step 4: Use the context
kubectl config use-context my-context
```

### Option 4: Skip kubeconfig Altogether

Instead of using a kubeconfig, you can set all the cluster details using CLI arguments and then use an alias for a clean terminal

#### For Token Authentication
Node env vars
```bash
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
CACERT=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
```

Pod env vars:
```bash
TOKEN=$(cat /run/secrets/kubernetes.io/serviceaccount/token)
CACERT=/run/secrets/kubernetes.io/serviceaccount/ca.crt
```

```bash
alias kubectl="kubectl --token=$TOKEN \
--server="https://kubernetes.default.svc.cluster.local" \
--certificate-authority=$CACERT"

kubectl get pods
```

#### For Certificate Authentication

```bash
kubectl  
alias kubectl="kubectl --client-certificate kubelet.crt \
--client-key kubelet.key \
--server="https://kubernetes.default.svc.cluster.local" \
--certificate-authority=$CACERT"

kubectl get pods
```