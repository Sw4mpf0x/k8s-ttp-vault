# Setup
Set the following environmental variables
```bash
export APISERVER=${KUBERNETES_SERVICE_HOST}:${KUBERNETES_SERVICE_PORT_HTTPS}
export SERVICEACCOUNT=/var/run/secrets/kubernetes.io/serviceaccount
export NAMESPACE=$(cat ${SERVICEACCOUNT}/namespace)
export TOKEN=$(cat ${SERVICEACCOUNT}/token)
export CACERT=${SERVICEACCOUNT}/ca.crt
alias kurl="curl --cacert ${CACERT} --header \
	\"Authorization: Bearer ${TOKEN}\" \
	\"Content-Type: application/json\""
# if kurl is still got cert Error, using -k option to solve this.
```
# Enumeration

List permissions in `default` namespace
```bash
kurl -X POST \
  $APISERVER/apis/authorization.k8s.io/v1/selfsubjectrulesreviews \
  -d '{
    "apiVersion": "authorization.k8s.io/v1",
    "kind": "SelfSubjectRulesReview",
    "spec": {
      "namespace": "default"
    }
  }' | jq .
```

Namespaces:
```bash
# List API endpoints
kurl https://$APISERVER$/api/v1/

# namespaces
kurl https://$APISERVER$/api/v1/namespaces/

# Secrets
kurl https://$APISERVER$/api/v1/namespaces/$NAMESPACE/secrets/

# Deployments
kurl https://$APISERVER$/api/v1/namespaces/$NAMESPACE/deployments/

# Pods
kurl https://$APISERVER$/api/v1/namespaces/$NAMESPACE/pods/

# Services
kurl https://$APISERVER$/api/v1/namespaces/$NAMESPACE/services/

# Nodes
kurl https://$APISERVER$/api/v1/nodes/

# Daemonsets
kurl https://$APISERVER$/api/v1/namespaces/$NAMESPACE/daemonsets/

# Secrets
kurl https://$APISERVER$/api/v1/namespaces/$NAMESPACE/secrets/
```
