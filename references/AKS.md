
# Identity

Azure Kubernetes Service (AKS) primarily uses Microsoft Entra ID (formerly Azure Active Directory) for authentication, which allows for centralized identity management and integration with Azure role-based access control (RBAC). This setup enables secure access to AKS clusters and supports features like multifactor authentication (MFA) and conditional access policies.

Regarding mixed mode authentication, AKS does not natively support a mixed mode that allows both Azure AD and non-Azure AD access simultaneously. However, you can use Kubernetes role-based access control (RBAC) to manage access within the cluster, which can be integrated with external identity solutions if needed. This integration allows for more traditional access methods for human administrators or developers, but it still relies on Azure AD for identity management in AKS clusters.

For more detailed information on AKS authentication and identity options, you can refer to the [Azure documentation on AKS identity](https://learn.microsoft.com/en-us/azure/aks/concepts-identity).
# Options for connecting to the private cluster  ([Source](https://learn.microsoft.com/en-us/azure/aks/private-clusters?tabs=default-basic-networking%2Cazure-portal#options-for-connecting-to-the-private-cluster))

The kube API server endpoint has no public IP address. To manage the API server, you need to use a VM that has access to the AKS cluster's Azure Virtual Network (VNet). There are several options for establishing network connectivity to the private cluster:

- Create a VM in the same VNet as the AKS cluster using the [`az vm create`](https://learn.microsoft.com/en-us/cli/azure/vm#az-vm-create) command with the `--vnet-name` flag.
- Use a VM in a separate network and set up [virtual network peering](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-network-peering-overview).
- Use an [Express Route or VPN](https://learn.microsoft.com/en-us/azure/expressroute/expressroute-about-virtual-network-gateways) connection.
- Use the [AKS `command invoke` feature](https://learn.microsoft.com/en-us/azure/aks/access-private-cluster).
- Use a [private endpoint](https://learn.microsoft.com/en-us/azure/private-link/private-endpoint-overview) connection.
- Use a [Cloud Shell](https://learn.microsoft.com/en-us/azure/cloud-shell/vnet/overview) instance deployed into a subnet that's connected to the API server for the cluster.

Creating a VM in the same VNet as the AKS cluster is the easiest option. Express Route and VPNs add costs and require additional networking complexity. Virtual network peering requires you to plan your network CIDR ranges to ensure there are no overlapping ranges.

# If you have node access...

AKS nodes (VMSS intances) have multiple identities, so you have to specify which identity to query for when trying to get an access token from the metadata API with the `client_id`, `object_id`, or `msi_res_id` parameters. You can find an initial client ID in the `userAssignedIdentityID` field  of the `/etc/kubernetes/azure.json` file

```bash
cat /etc/kubernetes/azure.json | jq .
```

* If `useManagedIdentityExtension` is `true`, the cloud provider is using Managed Identity.
* `userAssignedIdentityID` is typically the **client ID (GUID)** of the user-assigned MI expected by components. If it looks like `/subscriptions/.../resourceGroups/.../providers/Microsoft.ManagedIdentity/userAssignedIdentities/...`, that’s an **MSI resource ID** instead (older tooling sometimes used the resource ID). Either way, you can feed the value to IMDS using `client_id=` (for GUIDs) or `msi_res_id=` (for full ARM IDs). IMDS parameter names and usage are in Microsoft’s token docs.
* There might be client IDs in the `/var/log/containers` logs as well.

```bash
sudo grep -RinE 'userAssigned|identity|client.?id|msi' /var/log/containers | head -n 50
```

# Unique Pods and Operators
### csi-azurefile-node

A DaemonSet for the CSI driver for Azure Files. It runs on each node and handles the “node‐side” operations of mounting/unmounting Azure Files shares for pods. Essentially: when a pod asks for a volume backed by Azure Files, the controller provisions the share, and the node daemon handles actually attaching and mounting it into the node’s filesystem namespace.

AKS supports dynamic provisioning of Azure Files shares via the CSI driver. The “node” component ensures each node can mount/unmount volumes as pods schedule there.

Reference: [https://learn.microsoft.com/en-us/azure/aks/azure-files-csi](https://learn.microsoft.com/en-us/azure/aks/azure-files-csi)

[https://github.com/kubernetes-sigs/azurefile-csi-driver](https://github.com/kubernetes-sigs/azurefile-csi-driver)

### csi-azuredisk-node

Similar to the Azure Files case, this is the node‐side DaemonSet for the CSI driver for Azure Disk volumes. When a pod requests a block disk volume (Azure Disk) via PersistentVolumeClaim, the controller provisions that disk, and the node agent handles attaching/mounting it to the node.

Reference: [https://docs.azure.cn/en-us/aks/csi-storage-drivers](https://docs.azure.cn/en-us/aks/csi-storage-drivers)

[https://github.com/kubernetes-sigs/azuredisk-csi-driver](https://github.com/kubernetes-sigs/azuredisk-csi-driver)

### cloud-node-manager

A component of the Kubernetes node lifecycle controller that handles node registration, node updates, Azure VM/scale-set integration, etc.

[https://cloud-provider-azure.sigs.k8s.io/install/azure-ccm/](https://cloud-provider-azure.sigs.k8s.io/install/azure-ccm/)

[https://github.com/kubernetes-sigs/cloud-provider-azure/blob/master/examples/out-of-tree/cloud-node-manager.yaml](https://github.com/kubernetes-sigs/cloud-provider-azure/blob/master/examples/out-of-tree/cloud-node-manager.yaml)

### azure-ip-masq-agent

DaemonSet that runs on each node and configures iptables rules to handle IP masquerading (SNAT) for traffic leaving pods, when needed. Fork of upstream _ip-masq-agent_

# Unique Secrets

- `kube-system/omsagent-aad-msi-token` - Token for OMS Agent (Azure Monitor Agent) . Used to send logs/metrics to Log Analytics workspace
- `kube-system/aad-msi-auth-token` - scoped to monitor.azure.com
- `kube-system/ama-logs-secret` - Azure Monitor Agent credentials (newer version of OMS). Contains Workspace ID and shared key. likely has the **Log Analytics Workspace Shared Key** (Shared key = FULL workspace access)
- `node-manager/azure-storage-account-fd512e33afa504e1ab930c2-secret` - Azure Storage Account credentials. Used by node-manager (likely for CSI driver or persistent volumes)
- `kube-system/konnectivity-certs` - Client certificates for Konnectivity agent. Used for control plane to node communication tunneling.
- `kube-system/azure-policy-webhook-cert` - Azure Policy addon webhook certificates. Enforces Azure governance policies in cluster.
- `kube-system/microsoft-defender-ws-secret` - Microsoft Defender for Cloud credentials. Used to send security telemetry to Azure Security Center.
- `kube-system/ama-metrics-operator-targets-*-tls-secret` - TLS certificates for Azure Monitor metrics collection. Client/server certs for secure metric transmission.