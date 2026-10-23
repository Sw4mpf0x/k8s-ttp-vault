```bash
kubectl debug node/aks-nodepool1-37663765-vmss000000 -it --image=mcr.microsoft.com/cbl-mariner/busybox:2.0

# Access node filesystem
chroot /host

# Now you can run commands like:
systemctl status kubelet
journalctl -u kubelet
df -h
top

# cleanup
kubectl delete pod node-debugger-aks-nodepool1-37663765-vmss000000-bkmmx
```

Next: [[Node Reconnaissance]]