`oc` provides all of the functionality of the Kubernetes `kubectl` CLI tool. In addition, `oc` makes it easier to work with OpenShift. The CLI is ideal in situations where you are:

- Working directly with project source code
- Scripting OpenShift operations
- Cannot use the web console because of resource bandwidth restrictions

```bash
oc login -u admin -p admin https://api.crc.testing:6443 --insecure-skip-tls-verify=true  
oc get nodes  
oc get pods -A  
oc exec kube-apiserver-crc -n openshift-kube-apiserver -i -t -- bash  

# CSR Signer Keys (Cluster CA)  
ls /etc/kubernetes/static-pod-resources/kube-controller-manager-certs/secrets/csr-signer
```

OpenShift is often referred to as a container application platform in that it's a platform designed for the development and deployment of applications in [Linux containers](https://developers.redhat.com/topics/containers).

OpenShift has an organizational unit named **project**. You use a **project** to group resources in your application. The reason for organizing your application in a **project** is to enable controlled access and quotas for developers or teams.

You can think of a **project** as a visualization of the Kubernetes namespace based on the developer access controls.

Trying to run a debug pod:

```bash
oc debug node/crc
```