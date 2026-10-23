# Namespaces
## Component Indicative Namespace

If you see:
* `flux-system` -> [[FluxCD|FluxCD is deployed]]
* `argocd` -> ArgoCD is deployed
* `vault` - > HashiCorp Vault
* `gitlab-runner` -> A Gitlab runner is deployed 
## Platform Indicative Namespaces
AWS EKS:
* amazon-cloudwatch
* aws-observability

Azure AKS:
* gatekeeper-system
* tigera-operator

GCP GKE:
* gke-system
* `gke-managed-*`
* gmp-system

# Service Accounts
* default 
* cluster-admin
* admin 
* controller
* manager
* operator
* deployer