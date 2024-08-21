Helm install 
```bash
sudo dnf install -y git tar
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```


Calico Helm install
```bash
helm repo add projectcalico https://projectcalico.docs.tigera.io/charts
kubectl create namespace tigera-operator
helm show values projectcalico/tigera-operator

cat <<EOF | tee calicoValues.yaml
# imagePullSecrets is a special helm field which, when specified, creates a secret
# containing the pull secret which is used to pull all images deployed by this helm chart and the resulting operator.
# this field is a map where the key is the desired secret name and the value is the contents of the imagePullSecret.
#
# Example: --set-file imagePullSecrets.gcr=./pull-secret.json
imagePullSecrets: {}

# Configures general installation parameters for Calico. Schema is based
# on the operator.tigera.io/Installation API documented
# here: https://projectcalico.docs.tigera.io/reference/installation/api#operator.tigera.io/v1.InstallationSpec
installation:
  enabled: true
  kubernetesProvider: ""

  # imagePullSecrets are configured on all images deployed by the tigera-operator.
  # secrets specified here must exist in the tigera-operator namespace; they won't be created by the operator or helm.
  # imagePullSecrets are a slice of LocalObjectReferences, which is the same format they appear as on deployments.
  #
  # Example: --set installation.imagePullSecrets[0].name=my-existing-secret
  imagePullSecrets: []

# Configures general installation parameters for Calico. Schema is based
# on the operator.tigera.io/Installation API documented
# here: https://projectcalico.docs.tigera.io/reference/installation/api#operator.tigera.io/v1.APIServerSpec
apiServer:
  enabled: true

# Certificates for communications between calico/node and calico/typha.
# If left blank, will be automatically provisioned.
certs:
  node:
    key:
    cert:
    commonName:
  typha:
    key:
    cert:
    commonName:
    caBundle:

defaultFelixConfiguration:
  enabled: true
  iptablesBackend: NFT

# Resources for the tigera/operator pod itself.
# By default, no resource requests or limits are specified.
resources: {}

# Tolerations for the tigera/operator pod itself.
# By default, will schedule on all possible place.
tolerations:
- effect: NoExecute
  operator: Exists
- effect: NoSchedule
  operator: Exists

# NodeSelector for the tigera/operator pod itself.
nodeSelector:
  kubernetes.io/os: linux

# Custom annotations for the tigera/operator pod itself
podAnnotations: {}

# Custom labels for the tigera/operator pod itself
podLabels: {}

# Configuration for the tigera operator images to deploy.
tigeraOperator:
  image: tigera/operator
  registry: quay.io
calicoctl:
  image: docker.io/calico/ctl

# Configuration for the Calico CSI plugin - setting to None will disable the plugin, default: /var/lib/kubelet
kubeletVolumePluginPath: None

# Optionally configure the host and port used to access the Kubernetes API server.
kubernetesServiceEndpoint:
  host: ""
  port: "6443"

EOF

helm install -f calicoValues.yaml calico projectcalico/tigera-operator --namespace tigera-operator

helm status calico --namespace tigera-operator
helm ls -A
```
> REF: https://artifacthub.io/packages/helm/projectcalico/tigera-operator
