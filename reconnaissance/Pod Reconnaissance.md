---
tags:
  - pod
  - recon
---
#  Local Enumeration

* Local enumeration on a pod looks like typical linux-based enumeration
* Your pod may have more than one container running on it so there may be additional attack surface there
* Keep in mind that this pod likely has a deployed application running, so investigate the source code, mounted/env var secrets, and external or kubernetes resources that it uses and has access to

# Enumerate the Pod's Service Account

The pod's service account token, namespace, and cluster CA public cert are stored in the mounted folder:
```bash
/run/secrets/kubernetes.io/serviceaccount/token
/run/secrets/kubernetes.io/serviceaccount/namespace
/run/secrets/kubernetes.io/serviceaccount/ca.crt
```
See [[Service Accounts]] for more information
# Interact with the Kubernetes API
Typically, your next step after landing on a pod is to start enumerating the cluster with the pod's service account. Sometimes this is a dead end because your pod has no access, but sometimes you get lucky...
### Option 1: Proxy kubectl
Note that mounted service tokens are short-lived (typically an hour) and will need to be periodically retrieved from the pod. You can also [[Service Accounts#Create Persistent Service Account Token|Provision a long-term token]] if you have the permissions.
-> [[Setup kubectl#Assessment VM|Download and setup kubectl]]
Next step: [[Kube API Reconnaissance (kubectl)|Enumerate the K8s API with kubectl]]
### Option 2: Download kubectl
If the file system is writeable -> [[Setup kubectl#Download|Download and setup kubectl]]
If in a read-only filesystem -> [[#Finding a Writeable Mount|Find a writeable mount]]
Next step: [[Kube API Reconnaissance (kubectl)|Enumerate the K8s API with kubectl]]
### Option 3: Use curl
-> [[Kube API Reconnaissance (curl)|Enumerate the K8s API with curl]]

## Finding a Writeable Mount
If you land on a read-only file system, you might be able to find a mounted drive that you can write to.

```bash
mount | grep -v '\bro\b'
# or from the kernel's view:
findmnt -t noro
findmnt -o TARGET,FSTYPE,OPTIONS

# Filter out writable mounts with noexec
findmnt -o TARGET,OPTIONS | grep -v noexec
```

# Port Scanning

* By default, the cluster network is set to allow all pods to talk to each other. The **CNI plugin** installed on each node decides **what IP address a pod gets**, based on its configured **IP address pool (subnet)** and can also be used to implement network policies (i.e., firewall rules) between namespaces
	* NetworkPolicies can’t control access to:
		- **NodePort / LoadBalancer** services.
		- Pods using `hostNetwork: true` **bypass the pod network** and aren’t affected by NetworkPolicies.  
		- **Ingress controllers** or **host-level services**.
		- **DNS** or **host networked pods**.
* Clusters use a `/16` CIDR IP space, but in reality they use a small subset of it. Scan identified `/24` IP spaces instead

`nmap -T4 192.168.0.0/24 -oA scan -p 80,443,2379,4194,8080,9090,9100,9093,4001,6782,6783,6784,6443,8443,9099,10250,10255,10256 --open`

Kubernetes Port:
```
443/TCP		kube-apiserver		Kubernetes API port
2379/TCP		etcd
6666/TCP		etcd				etcd
4194/TCP		cAdvisor			Container metrics
6443/TCP		kube-apiserver		Kubernetes API port
8443/TCP		kube-apiserver		Minikube API port
8080/TCP		kube-apiserver		Insecure API port
10250/TCP		kubelet				HTTPS API which allows full mode access
10255/TCP		kubelet				Unauthenticated read-only HTTP port: pods, running pods and node state
10256/TCP		kube-proxy			Kube Proxy health check server
9099/TCP		calico-felix		Health check server for Calico
6782-4/TCP      weave				Metrics and endpoints
```

If you see port 10250 or 10255, that is the [[kubelet|kubelet API hosted on a node]]

Query kubelet API:
```bash
curl -sk http://<nodeip>:4194/metrics
```

etcd:
```bash
curl -k https://<IP address>:2379
curl -k https://<IP address>:2379/version
```

If you can reach etcd: [[etcd|enumerate it]]

# Metadata API

If your pod is running in a cloud provider, try to access your node's metadata API for cloud compute or kubelet credentials

### GCP

Grab kubelet certificate
```bash
`curl -s -H 'Metadata-Flavor: Google' 'http://metadata.google.internal/computeMetadata/v1/instance/attributes/kube-env'`
```

### AWS

On AWS, you will need a pod with `hostNetork`  or a `Privileged`  pod to access to the Node’s metadata API.

## Azure

See [[AKS#If you have node access...|AKS metadata API guidance]]