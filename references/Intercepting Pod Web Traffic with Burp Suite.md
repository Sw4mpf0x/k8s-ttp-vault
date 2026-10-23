## 1. Export and encode the Burp CA

In Burp: Proxy → Options → CA Certificate → Export → DER format → save as `burp.der`
```bash
# Convert DER to PEM
openssl x509 -inform DER -in burp.der -out burp.pem

# base64-encode (macOS — strip newlines)
BURP_CA_B64=$(cat burp.pem | base64 | tr -d '\n')
echo $BURP_CA_B64   # copy this value into the ConfigMap below
```

Verify Burp is reachable from a node. This should return Burp's welcome page HTML:
```bash
docker exec <node-name> curl -v --max-time 3 http://<mac-lan-ip>:8080
```

## 2. Inject CA into Node Trust Stores

Get a list of node names with:
```bash
kubectl get nodes -o wide
```

Plug the node names in and run the following to push the Burp pem certificate to all nodes:
```bash
PEM=$(echo $BURP_CA_B64 | base64 -d)

for NODE in <control-plane-node> <worker-node> <worker2-node>...; do
  echo "=== $NODE ==="
  echo "$PEM" | docker exec -i "$NODE" sh -c '
    BUNDLE=/etc/ssl/certs/ca-certificates.crt
    STANDALONE=/etc/ssl/certs/burp-suite-proxy.crt
    MARKER="Burp Suite Proxy CA"
    cat > "$STANDALONE"
    if ! grep -qF "$MARKER" "$BUNDLE" 2>/dev/null; then
      printf "\n# %s\n" "$MARKER" >> "$BUNDLE"
      cat "$STANDALONE" >> "$BUNDLE"
      echo "  [OK] appended"
    else
      echo "  [SKIP] already present"
    fi
  '
done
```

Verify:
```bash
for NODE in <control-plane-node> <worker-node> <worker2-node>; do
  echo -n "$NODE: "
  docker exec "$NODE" grep -c "Burp Suite Proxy CA" /etc/ssl/certs/ca-certificates.crt
done
# Each should return 1
```

### 3. Inject Proxy Env Vars Into Pods

We need to inject `HTTP_PROXY`, `HTTP_PROXYS`, and `NO_PROXY` environmental variables into the target operator's pod so it sends web traffic to Burp Suite. We can use Kyverno to add them to all pods deployed to a namespace, or we can directly modify the target operator's deployment. 

`NO_PROXY` construction rationale:

| Excluded value                                                           | Reason                                                                                                                                                      |
| ------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `kubernetes.default.svc`, <br>`kubernetes`, <br>`.svc`, `.cluster.local` | kube-apiserver calls via Service DNS — if routed through Burp, it presents a cert with no IP SAN for 10.96.0.1, causing fatal x509 error blocking CRD watch |
| `10.0.0.0/8`                                                             | Covers pod CIDR (10.244.x.x) and service CIDR (10.96.x.x)                                                                                                   |
| `172.16.0.0/12`                                                          | Docker bridge range                                                                                                                                         |
| `localhost`, `127.0.0.1`                                                 | Loopback                                                                                                                                                    |
#### Option  1: Kyverno Webhook

Install Kyverno:
```bash
kubectl create -f https://github.com/kyverno/kyverno/releases/download/v1.16.2/install.yaml
```

Apply the following policy:
```yaml
apiVersion: kyverno.io/v1
kind: Policy
metadata:
  name: inject-proxy-env
  namespace: <namepspace>
spec:
  rules:
    - name: inject-proxy-env-vars
      match:
        any:
          - resources:
              kinds:
                - Pod
      mutate:
        patchStrategicMerge:
          spec:
            containers:
              - (name): "?*"
                env:
                  - name: HTTP_PROXY
                    value: "http://172.18.0.1:8081"
                  - name: HTTPS_PROXY
                    value: "http://172.18.0.1:8081"
                  - name: NO_PROXY
                    value: ".local,localhost,127.0.0.1,::1,kubernetes.default.svc,kubernetes,.svc,.cluster.local,10.0.0.0/8,172.16.0.0/12"
            initContainers:
              - (name): "?*"
                env:
                  - name: HTTP_PROXY
                    value: "http://172.18.0.1:8081"
                  - name: HTTPS_PROXY
                    value: "http://172.18.0.1:8081"
                  - name: NO_PROXY
                    value: ".local,localhost,127.0.0.1,::1,kubernetes.default.svc,kubernetes,.svc,.cluster.local,10.0.0.0/8,172.16.0.0/12"
```
#### Option 2: Direct Deployment Modification

##### Scale Down Reconciler

If there is a reconciler that enforces infrastructure as code (IaC) in the cluster, you will need to scale down its deployment because we will be modifying the target operator's deployment and this will overwrite our modified deployment with its IaC manifest.

```bash
# Step 1: Freeze reconciliation
kubectl scale deployment/fluxcd \
  -n <namespace> --replicas=0

# Confirm reconciler is stopped
kubectl rollout status deployment/fluxcd \
  -n <namespace>
```

##### Modify the Target Deployment

```bash
# Find the target deployment
kubectl get deployments -n <namespace>

# Patch the deployment
kubectl patch deployment my-agent \
  -n <namespace> \
  --type=json \
  -p='[
    {"op": "add", "path": "/spec/template/spec/containers/0/env/-",
     "value": {"name": "HTTPS_PROXY", "value": "http://<mac-lan-ip>:8080"}},
    {"op": "add", "path": "/spec/template/spec/containers/0/env/-",
     "value": {"name": "HTTP_PROXY", "value": "http://<mac-lan-ip>:8080"}},
    {"op": "add", "path": "/spec/template/spec/containers/0/env/-",
     "value": {"name": "NO_PROXY",
               "value": ".local,localhost,127.0.0.1,::1,kubernetes.default.svc,kubernetes,.svc,.cluster.local,10.0.0.0/8,172.16.0.0/12"}}
  ]'

# Wait for rollout
kubectl rollout status deployment/my-agent -n <namespace> --timeout=120s
```