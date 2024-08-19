> Please see previous steps on the root [readme](./readme.md)

# Deploy the Calico Container Network Interface (CNI)
The `kubeadm init` output instructs that you must also deploy a pod network, a Container Network Interface (CNI), to get things working. Base Kubernetes is tested with Calico, and supports IPv6.

On the CONTROL node prep the cluster with Calico custom resource definitions
> At time of writing, the Calico version is 3.28.1.  Please adjust as needed, but PLEASE NOTE there are breaking changes between versions and compatability requirements with Calico.
```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.1/manifests/tigera-operator.yaml
```

> References
> https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart
> https://docs.tigera.io/calico/latest/networking/ipam/ipv6#enable-ipv6-only

### <a name="calicovxlan" >OPTION: Calico with VXLAN
This EASY network option allows you to quickly get connectivity of `pods` between `nodes`.   This does NOT provide network access directly to the pods or services.   This mimics the isolated 'internal' network of a Docker instance.   With a Cloud provider, they provide a `LoadBalancer` into this network.   A future write-up may address this.  If you want external access to the pod network, use the [OPTION: Calico with BGP](OPTION:-Calico-with-bgp) below.

On the CONTROL node create a `.yaml` file for initializing the CNI.  A modified configuration file with the IPv6 changes already applied is below and available for download. </br>
https://github.com/ziptx/kubernetes-ipv6/blob/4b1c2fc53db76b298a5b85e943b5ec9278ccb764/calico-basic-vxlan.yaml#L1-L22
-OR- Download the file
```bash
curl -O https://raw.githubusercontent.com/ziptx/kubernetes-ipv6/main/calico-basic-vxlan.yaml
```

[embedmd]:# (calico-basic-vxlan.yaml yaml)


On the CONTROL node apply the '.yaml' configuration file from above:
```bash
kubectl create -f calico-basic-vxlan.yaml
```

After this you can check everything is running. If successful, then there should be some calico pods, and the coredns pods should now be running.

```bash
watch kubectl get pods -n calico-system
```

> [!TIP]
> Some troubleshooting commands:
> ```
>  kubectl get tigerastatus -o yaml
> ```

Skip to [Install calicoctl](install-calicoctl) to continue


### OPTION: Calico with BGP

To be added

### Install calicoctl
`calicoctl` is a CLI tool that is largely meant to be run on a user's client machine and shouldn't need to be deployed to the cluster itself, and the operator is responsible for installing Calico components that run on the cluster.  `calicoctl` is being depricated but is required to manage Calico BGP installs.  The following assumes administrations is being done on the CONTROL node, and is installed there. 

> [!TIP]
> Consider navigating to a location that's in your PATH for install. For example, /usr/local/bin/.

> [!TIP]
> Much documentation will refer to using `calicoctl`, but the plugin can also be used as `kubectl calico`.  ie `kube calico -h`  The single host instructions below setup both commands to work.

#### Install calicoctl as a binary on a single host
https://docs.tigera.io/calico/latest/operations/calicoctl/install#install-calicoctl-as-a-binary-on-a-single-host
> At time of writing, the Calico version is 3.28.1.  Please adjust as needed, but PLEASE NOTE there are breaking changes between versions and compatability requirements with Calico.
```bash
cd /usr/local/bin
sudo curl -L https://github.com/projectcalico/calico/releases/download/v3.28.1/calicoctl-linux-amd64 -o kubectl-calico
sudo chmod +x kubectl-calico
cat <<EOF | sudo tee /usr/local/bin/calicoctl
/usr/local/bin/kubectl-calico "\$@"
EOF
sudo chmod +x calicoctl
```
