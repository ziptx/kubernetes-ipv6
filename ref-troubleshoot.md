## Version of IPTables/NFTables
```bash
kubectl get pods -n calico-system
kubectl exec -it -n calico-system calico-node-xx -- iptables -V
# answer of (legacy) is bad
kubectl logs -n calico-system calico-node-xx  -c calico-node | grep iptablesbackend
2023-07-30 03:01:52.069 [INFO][15] tunnel-ip-allocator/env_var_loader.go 40: Found felix environment variable: "iptablesbackend"="NFT"
```
