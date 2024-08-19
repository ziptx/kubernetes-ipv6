## Version of IPTables/NFTables
```bash
kubectl get pods -n calico-system
kubectl exec -it -n calico-system calico-node-xx -- iptables -V
# answer of (legacy) is bad
kubectl logs -n calico-system calico-node-xx  -c calico-node | grep iptablesbackend
2023-07-30 03:01:52.069 [INFO][15] tunnel-ip-allocator/env_var_loader.go 40: Found felix environment variable: "iptablesbackend"="NFT"
```

## Add IPTables Blacklist
```bash
sudo cat <<EOF | sudo tee /etc/modprobe.d/10-blacklist-iptables.conf
blacklist ip_tables
EOF
sudo dracut -f
```
> https://mihail-milev.medium.com/no-pod-to-pod-communication-on-centos-8-kubernetes-with-calico-56d694d2a6f4
