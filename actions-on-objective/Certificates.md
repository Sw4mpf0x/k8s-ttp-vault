
# Certificate Subject Format

The string `/CN=kube-admin/O=system:masters` is a **Distinguished Name (DN)** from an X.509 certificate, which Kubernetes uses for authentication and authorization.

## Breaking Down the Components:

- **`/CN=kube-admin`** - **Common Name**
  - This becomes the **username** in Kubernetes
  - Used to identify the specific user or service
- **`/O=system:masters`** - **Organization**
  - This becomes the **group membership** in Kubernetes
  - Users can belong to multiple groups (multiple `/O=` entries)

## Key Kubernetes Naming Conventions

### **System Groups** (prefix: `system:`)

```
system:masters           # Cluster superusers. hardcoded in Kubernetes to have cluster-admin privileges - it bypasses all RBAC checks and has full cluster access.
system:nodes             # Kubelet nodes
system:serviceaccounts   # All service accounts
system:authenticated     # All authenticated users
system:unauthenticated   # Unauthenticated requests
```

### **System Users** (prefix: `system:`)

```
system:admin             # Administrative user
system:kube-scheduler    # Scheduler component  
system:kube-controller-manager
system:node:node-name    # Specific node identity
```

### **Service Account Format**

```
system:serviceaccount:namespace:name
# Example: system:serviceaccount:kube-system:default
```


# Golden Certificate

| Cluster Type                | CA Private Key Access | Location                                                                         |
| --------------------------- | --------------------- | -------------------------------------------------------------------------------- |
| **AKS**                     | ❌ No access           | Managed by Azure (HSM)                                                           |
| **EKS**                     | ❌ No access           | Managed by AWS                                                                   |
| **GKE**                     | ❌ No access           | Managed by Google                                                                |
| **OpenShift (managed)**     | ❌ Limited/No access   | Managed by Red Hat                                                               |
| **OpenShift (self-hosted)** | ⚠️ Restricted access  | `/etc/kubernetes/` on masters, or secrets in `openshift-kube-controller-manager` |
| **kubeadm**                 | ✅ Full access         | `/etc/kubernetes/pki/ca.key`                                                     |
| **k3s**                     | ✅ Full access         | `/var/lib/rancher/k3s/server/tls/`                                               |
| **kind/minikube**           | ✅ Full access         | `~/.minikube/ca.key` or in container                                             |

# Certificate Signing Requests (CSR)

Generate a certificate and CSR 
```bash
# 1. Generate a private key
openssl genrsa -out user.key 2048

# 2. Create a certificate signing request
openssl req -new -key user.key -out user.csr -subj "/CN=<username>/O=system:masters"
```

Submit your CSR to Kubernetes:
```yaml
apiVersion: certificates.k8s.io/v1beta1
kind: CertificateSigningRequest
metadata:
  name: my-csr-$(date +%s)
spec:
  groups:
  - system:masters
  request: $(cat user.csr | base64 | tr -d '\n')
  usages:
  - digital signature
  - key encipherment
  - client auth
```

```bash
# Submit CSR
kubectl create -f csr.yml

# Check CSR
kubectl get certificatesigningrequests
```

A user with CSR approval access cant approve the request:
```bash
kubectl certificate approve my-csr-?
kubectl certificate deny my-csr-?
kubectl delete csr my-csr-?
```

After approval, Kubernetes signs the CSR with the CA key pair, and the certificate is embedded in the CertificateSigningRequest object's YAML output as a base64 encoded string.
Example:
```
...snipped...
status:
  certificate: L$0tL1CRUdJTiBDRVJUSUZJQ0FURS9tL0t1SURDakNDQWZLZ0F3SUJBZ0lVRmwyQ2wXYXoxalW5M3JNVisreFRYQYouW3dnd0RWpL1pJaHZjTkFRRUwkQkFBd0ZVUnRVMVhQTFRUVF4TUhM1ZpHkdVpMjxkF1RncweE9UQ1NVE14TmpNeU1EQmFgdGl0dY0ZFBl2ajNuSXY3eFd3I1Rm5u440c0t520vXukwTFM5V29ge1hHZdWCMlEZ2FOMVVMRFBXTVhjN09FVnVjSk1k4weRUVtR5tD11zWeHVjS1h6g1dV0pMediMUGbXYFKWVKWMVmBjRVRTY3dod2xiO1ND0kLS0tL1F0kQg0V5VElSGUNBVEUt
  conditions:
    - lastUpdateTime: 2019-02-13T16:37:21Z
      message: This CSR was approved by kubectl certificate approve.
      reason: KubectlApprove
      type: Approved
```

Retrieve new certificate:
```bash
# Get private cert from approved CSR
kubectl get certificatesigningrequests my-csr-1588631615 -o jsonpath='{.status.certificate}' | base64 -d > user.crt

# Test it out
kubectl --client-certificate user.crt --client-key user.key auth can-i --list -n kube-system
```

See [[Setup kubectl#For Certificate Authentication|Certificate Authentication]] to setup cert-based authentication in kubectl 