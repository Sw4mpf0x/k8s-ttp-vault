* Network policies are tied to pods via a label selector and can be applied to ingress and egress
* By default, the cluster is a flat network, so network policies allow admins to control pod-to-pod and pod-to-external access
* Once any NetworkPolicy is applied to a Pod, all traffic not explicitly allowed is denied
* Namespace-scoped
* Enforced by a CNI plugin (Calico, Cilium, OVN, etc.)
* Network Policies cannot apply to:
	* Any pod with Node host network access (i.e., `hostNetwork: true`)
	* NodePort, LoadBalancer, or Ingress Controller traffic before it hits Pods
	* DNS (depending on CNI)
	* Non-TCP protocols 
	* Cluster API access (due to `hostNetwork: true`)
	* Service Mesh Sidecars


Example NetworkPolicy
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          name: api-pod
    ports:
    - protocol: TCP
      port: 3306
```
# Cilium

Cilium is a Container Network Interface (CNI) that uses `CiliumNetworkPolicy` and `CiliumClusterwideNetworkPolicy` CRDs to store network policies. 

You can audit them by querying kubectl:
```bash
kubectl get cnp -n <namespace> -o yaml 
kubectl get cnp -A -o yaml
```

You can also use [CiliumHound](https://github.com/Sw4mpf0x/CiliumHound) to graph `CiliumNetworkPolicy` policies that you dumped with:
```bash
kubectl get cnp -A -o yaml > cnp-all.yaml
```

