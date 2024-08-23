## Version of IPTables/NFTables
```bash
kubectl get pods -n calico-system
kubectl exec -it -n calico-system calico-node-xx -- iptables -V
# answer of (legacy) is bad
kubectl logs -n calico-system calico-node-xx  -c calico-node | grep iptablesbackend
2023-07-30 03:01:52.069 [INFO][15] tunnel-ip-allocator/env_var_loader.go 40: Found felix environment variable: "iptablesbackend"="NFT"

kubectl -n calico-system describe pod calico-node-dtx8s | fgrep FELIX
```

## Add IPTables Blacklist
```bash
sudo cat <<EOF | sudo tee /etc/modprobe.d/10-blacklist-iptables.conf
blacklist ip_tables
EOF
sudo dracut -f
```
> https://mihail-milev.medium.com/no-pod-to-pod-communication-on-centos-8-kubernetes-with-calico-56d694d2a6f4

## Get BGP Tables from Calico Node
```bash
kubectl exec -n calico-system calico-node-x -- calico-node -show-status
calicoctl get bgpconfig
kubectl exec -it -n calico-system calico-node-x --
```

## Get Cluster IP Info & Test with BusyBox
```bash
kubectl apply -f https://k8s.io/examples/admin/dns/busybox.yaml
kubectl exec -ti busybox -- nslookup kubernetes.default

kubectl get pods --all-namespaces -o   custom-columns='NAMESPACE:metadata.namespace','NAME:metadata.name','STATUS:status.phase','IP:status.podIP'

ping -c 2 `kubectl get pods busybox -o=custom-columns='IP:status.podIP' --no-headers`
kubectl exec -ti busybox -- ping -c 2 `hostname -i`
kubectl exec -ti busybox -- ping -c 2   `kubectl get pods -n kube-system -o=custom-columns='NAME:metadata.name','IP:status.podIP' | grep -m 1 coredns | awk '{print $2}'`
```

## Deploy Busybox pods to test
This spins up (2) pods of busybox on each worker node.   So you can test pod<->pod connectivity locally on the node as well as across nodes.
```bash
#start test pods
kubectl create deployment pingtest --image=busybox --replicas=6 -- sleep infinity
#test between pods
kubectl get pods --selector=app=pingtest --output=wide
kubectl exec -ti pingtest-xxFromOutputAbovexx -- sh
#cleanup test pods
kubectl delete deployment pingtest
```
