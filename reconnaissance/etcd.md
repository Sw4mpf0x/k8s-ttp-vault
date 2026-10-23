
# Basics

etcd config file: `/etc/kubernetes/manifests/etcd.yaml`

etcd structure
* Pods: `/registry/pods/<namespace>/<name>`
* Deployments: `/registry/deployments/<namespace>/<name>`
* Services: `/registry/services/specs/<namespace>/<name>`
* ConfigMaps: `/registry/configmaps/<namespace>/<name>`
* Secrets: `/registry/secrets/<namespace>/<name>`
# CLI Setup
```bash
# Setup CLI
export ETCDCTL_API=3
export ETCD_ENDPOINT=https://127.0.0.1:2379
export CACERT=/etc/kubernetes/pki/etcd/ca.crt
export CERT=/etc/kubernetes/pki/etcd/server.crt
export KEY=/etc/kubernetes/pki/etcd/server.key

# Basic command
etcdctl --endpoints="$ETCD_ENDPOINT" --cacert="$CACERT" --cert="$CERT" --key="$KEY" member list
# or alias
alias etcdctl='etcdctl --endpoints="$ETCD_ENDPOINT" --cacert="$CACERT" --cert="$CERT" --key="$KEY"'
etcdctl member list
```

# Finding the etcd pod

Find the etcd pod and exec into it
```bash
# Find the etcd pod name on this node
kubectl -n kube-system get pods -o wide | grep ^etcd-
kubectl -n kube-system exec -it "$POD" -- sh
```
# etcd Enumeration
```bash
# Basic health & membership
etcdctl endpoint health
etcdctl member list

# Endpoint status (shows DB size, leader, etc.)
etcdctl endpoint status --write-out=table

# General usage
## List keys
etcdctl get "" --prefix --keys-only

## Get specific key
etcdctl get /registry/namespaces/default -w json

## Dump all key-value pairs
etcdctl get "" --prefix -w json > etcd_dump.json

# Explore Kubernetes data (keys only)
# Kubernetes uses etcd keys under /registry/... to store cluster state
etcdctl get / --prefix --keys-only | head -n 50

# List kube resources by prefix
etcdctl get /registry/pods --prefix --keys-only | head
etcdctl get /registry/deployments --prefix --keys-only | head
etcdctl get /registry/services --prefix --keys-only | head

# Fetch a specific object (replace NAMESPACE/NAME)
etcdctl get /registry/pods/NAMESPACE/NAME -w fields
etcdctl get /registry/pods/kube-system/etcd-al-control-plane -w json|jq '.kvs[0]'
etcdctl get /registry/pods/kube-system/etcd-al-control-plane -w json|jq '.kvs[0].value' | sed 's/"//g'| base64 -d

# Count how many kube objects are stored
etcdctl get /registry --prefix --keys-only | wc -l

# See all namespaces recorded
etcdctl get /registry/namespaces --prefix --keys-only

# Backup DB for offline analysis
etcdctl snapshot save /path/to/snapshot.db
# Verifies and summarizes the snapshot contents (revision, size, etc.).
etcdctl snapshot status /path/to/snapshot.db --write-out=table
```

# etcd Backup

## Restore a backup offline for analysis

Install `etcdctl`
```bash
# Download latest release (adjust version as needed)
ETCD_VER=3.5.15
GOOGLE_URL=https://storage.googleapis.com/etcd
DOWNLOAD_URL=${GOOGLE_URL}

wget ${DOWNLOAD_URL}/v${ETCD_VER}/etcd-v${ETCD_VER}-linux-amd64.tar.gz
tar xzvf etcd-v${ETCD_VER}-linux-amd64.tar.gz
sudo mv etcd-v${ETCD_VER}-linux-amd64/etcd* /usr/local/bin/
etcdctl version
```

Restore snapshot
```bash
etcdctl snapshot restore ~/snapshot.db --data-dir ~/etcd-restore
```

Run etcd service
```bash
etcd --data-dir ~/etcd-restore \
     --listen-client-urls http://127.0.0.1:2379 \
     --advertise-client-urls http://127.0.0.1:2379
```

Query it
```bash
export ETCDCTL_API=3
export ETCDCTL_ENDPOINTS=http://127.0.0.1:2379

# List keys
etcdctl get "" --prefix --keys-only
```