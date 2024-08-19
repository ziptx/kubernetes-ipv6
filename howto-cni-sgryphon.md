> Please see previous steos on the root [readme](../readme.md) </br>
> Refrence original writeup by [sgryphon](https://github.com/sgryphon/kubernetes-ipv6) 


## Deploy a Container Network Interface (CNI)

The `kubeadm init` output instructs that you must also deploy a pod network, a Container Network Interface (CNI), to get things working, and the output of init provides a link to a list. https://kubernetes.io/docs/concepts/cluster-administration/addons/

Base Kubernetes is tested with Calico, and it also supports IPv6, with instructions for the needed setup changes. https://docs.projectcalico.org/networking/ipv6

A modified configuration file with the IPv6 changes already applied is available for download. (Alternatively you can make the modifications yourself, see below.)

```bash
curl -O https://raw.githubusercontent.com/sgryphon/kubernetes-ipv6/main/calico-ipv6.yaml

sed -i 's/2001:db8:1234:5678/xxxx:xxxx:xxxx:xxxx/g' calico-ipv6.yaml
```

Then apply the configuration, which will set up all the custom object definitions and the calico service:

```bash
kubectl apply -f calico-ipv6.yaml
```

After this you can check everything is running. If successful, then there should be some calico pods, and the coredns pods should now be running.

```bash
kubectl get all --all-namespaces
```

### Installing calicoctl

For future administration of calico, you can also install calicoctl. https://docs.projectcalico.org/getting-started/clis/calicoctl/install

For example, to install calicoctl as a Pod:

```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calicoctl.yaml

# Save an alias in bash config, and then load it
cat <<EOF | tee -a ~/.bash_aliases
alias calicoctl="kubectl exec -i -n kube-system calicoctl -- /calicoctl"
EOF
. ~/.bash_aliases

calicoctl get profiles -o wide
```

### Aside: Manual modifications to the calico install file

An already modified version is available for download as above, otherwise you can make the changes yourself (e.g. if calico updates the main manifest).

Following the main install guide first, download the config file for "Install Calico with Kubernetes API datastore, 50 nodes or less".

```bash
curl https://docs.projectcalico.org/manifests/calico.yaml -O
```

Edit as per the IPv6 instructions and disable encapsulation (not needed for IPv6).

For step 3, modify the section at the top for "# Source: calico/templates/calico-config.yaml" in data > cni_network_config, and set assign_ipv6.

```json
 "ipam": {
        "type": "calico-ipam",
        "assign_ipv4": "false",
        "assign_ipv6": "true"
    },
```

For steps 4, 5, and the optional bit to turn off IPv4, you need to go past all the CustomResourceDefinition sections and find the section "# Source: calico/templates/calico-node.yaml".

Scroll down a bit more to find spec > template > spec > containers > name: calico-node > env, and set the environment variables. Enable IPv6 support, disable IPv4, set the router ID method. Also disable IPIP, as encapsulation is not needed with IPv6.

```yaml
- name: IP6
  value: "autodetect"
- name: FELIX_IPV6SUPPORT
  value: "true"
- name: IP
  value: "none"
- name: CALICO_ROUTER_ID
  value: "hash"
- name: CALICO_IPV4POOL_IPIP
  value: "Never"
- name: CALICO_IPV6POOL_CIDR
  value: "2001:db8:1234:5678:8:2::/104"
- name: CALICO_IPV6POOL_BLOCK_SIZE
  value: "120"
```

The documents say that Calico will pick up the IP pool from kubeadm, but it didn't work for me and I got the default CIDR `fda0:95bd:f195::/48`(a random `/48`) with the default block size `/122` (which is 64 pods per node), as per https://docs.projectcalico.org/reference/node/configuration


## Configure routing

Once the networking is configured you should have all the cluster components talking to each other.

You can test this by installing a Pod running a basic terminal instance (the Kubernetes examples use `busybox`, but also `dnsutils` and full `alpine`).

```bash
kubectl apply -f https://k8s.io/examples/admin/dns/busybox.yaml
kubectl exec -ti busybox -- nslookup kubernetes.default
```

You can check communication (ping) between the host and Pods, and between Pods:

```bash
kubectl get pods --all-namespaces -o \
  custom-columns='NAMESPACE:metadata.namespace','NAME:metadata.name','STATUS:status.phase','IP:status.podIP' 
ping -c 2 `kubectl get pods busybox -o=custom-columns='IP:status.podIP' --no-headers`
kubectl exec -ti busybox -- ping -c 2 `hostname -i`
kubectl exec -ti busybox -- ping -c 2 \
  `kubectl get pods -n kube-system -o=custom-columns='NAME:metadata.name','IP:status.podIP' | grep -m 1 coredns | awk '{print $2}'`
```

However communication with the rest of the Internet won't work unless incoming routes have been configured.

Your hosting provider may route the entire /64 to your machine, or may simply make the range available to you and be expecting your server to respond to Neighbor Discovery Protocol (NDP) solicitation messages with advertisement of the addresses it can handle.

Simply adding the address to the host would advertise it, but also handle the packets directly (INPUT), rather than forwarding to the cluster. To advertise the cluster addresses in response to solicitation you need to use a Neighbor Discovery Protocol proxy.

Linux does have some limited ability to proxy NDP, but it is limited to individual addresses; a better solution is to use `ndppd`, the Neighbor Discovery Protocol Proxy Daemon, which supports proxying of entire ranges of addresses, https://github.com/DanielAdolfsson/ndppd

This will respond to neighbor solicitation requests with advertisement of the Pod and Service addresses, allowing the upstream router to know to send those packets to the host. Use your addresses ranges in the configuration, advertising the Pod `/104` range automatically based on known routes, which includes individual pods (from `ip -6 route show`), and the entire `/112` Service range (as a static range).

```bash
# Install
sudo apt update
sudo apt install -y ndppd

# Create config
cat <<EOF | sudo tee /etc/ndppd.conf
proxy eth0 {
    rule 2001:db8:1234:5678:8:2::/104 {
        auto
    }
    rule 2001:db8:1234:5678:8:3::/112 {
        static
    }
}
EOF

# Restart
sudo systemctl daemon-reload
sudo systemctl restart ndppd

# Set to run on boot
sudo systemctl enable ndppd
```

Once running you can test that communication works between Pods and the Internet:

```bash
kubectl exec -ti busybox -- ping -c 2 2001:4860:4860::8888
```

### Routing alternatives

If `ndppd` does not suite your needs, then you can look at the basic NDP proxy support in Linux, or at the BGP support in Calico.

#### Basic NDP proxy

Linux directly supports NDP proxying for individual addresses, although the functionality is considered deprecated:

```
sysctl -w net.ipv6.conf.eth0.proxy_ndp=1
ip -6 neigh add proxy 2001:db8:1234:5678:8:2:42:1710 dev eth0
```

#### BGP

Calico support BGP, and runs full mesh BGP between nodes in the cluster. See https://docs.projectcalico.org/networking/bgp

For dynamic communication of routes to upstream routers, you can peer Calico using BGP (Note: based on documentation; not tested). Replace the values with those for your hosting provider, if they allow it.

```bash
cat <<EOF | tee calico-bgppeer.yaml
apiVersion: projectcalico.org/v3
kind: BGPPeer
metadata:
  name: global-peer
spec:
  peerIP: 2001:db1:80::3
  asNumber: 64123
EOF

calicoctl create -f calico-bgppeer.yaml

calicoctl node status
```
