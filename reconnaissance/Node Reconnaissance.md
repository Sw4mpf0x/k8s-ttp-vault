## Nodes Files of Interest

```bash
# kubelet credentials
cat /var/lib/kubelet/kubeconfig
cat /var/lib/kubelet/config.yaml
cat /etc/kubernetes/kubelet.conf
cat /var/lib/kubelet/pki/kubelet-client-current.pem

# Node's service account token and CA cert
ls /var/run/secrets/kubernetes.io/serviceaccount/

# etcd manifest if running on the node
cat /etc/kubernetes/manifests/etcd.yaml

# Node/Cluster info
ls /var/run/cloud-init/

# Other filesystems
ls -la /var/lib/docker/overlay2/
ls -la /var/lib/containerd/
```

##  Access Container Runtimes
`crictl` is natively available on nodes

```bash
# List all containers
crictl ps 
# Exec into any container
crictl exec -it <container-id> sh
# print a container's envvars
crictl inspect <container-id> | jq '.info.runtimeSpec.process.env' 
```

# Read Mounted Pod Secrets

* Nodes only received secrets for the pods they have deployed
*  **Secrets are mounted as tmpfs volumes** at `/var/lib/kubelet/pods/<pod-uid>/volumes/kubernetes.io~secret/`
* Each secret volume gets its own tmpfs mount

```bash
find /var/lib/kubelet/pods -name "secret" -type d
ls /var/lib/kubelet/pods/*/volumes/kubernetes.io~secret/*/*
cat /var/lib/kubelet/pods/*/volumes/kubernetes.io~secret/*/*

ls /var/lib/kubelet/pods/*/volumes/kubernetes.io~projected/kube-api*/
```