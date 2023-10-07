# Install Rancher 2.7.5 on Ubuntu 20.04
> You can find more information  about [Deploy HA k3s with kube-vip and MetalLB using k3sup](https://gist.github.com/rosswf/e5c4c85efb54a9f8d6e19a21cf09aa63).
> You can find more information  about [Simple RKE2, Longhorn, and Rancher Install](https://ranchergovernment.com/blog/article-simple-rke2-longhorn-and-rancher-install).

# 1. Install kubectl

```bash
apt update && apt upgrade -y && apt install -y wget curl jq vim net-tools telnet git open-vm-tools
curl -LO "https://dl.k8s.io/release/v1.26.7/bin/linux/amd64/kubectl"
install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

```

# 2. Install k3sup


```bash
curl -sLS https://get.k3sup.dev | sh

# install k3sup /usr/local/bin/   #if needed

# Install master node

k3sup install --local --ip <IP_NODE_RANCHER> --tls-san  <VIRTUAL_IP_1> --cluster --k3s-channel stable --k3s-version v1.26.7+k3s1 --k3s-extra-args "--disable servicelb" --local-path $HOME/.kube/config

cp /etc/rancher/k3s/k3s.yaml /root/.kube/config
```
> **Note:** IP_NODE_RANCHER is the IP of Rancher node.
> **Note:** VIRTUAL_IP_1 is the virtual IP .
# 3. Install Kube-VIP
> You can find more information about [**Kube-VIP**](https://kube-vip.io/docs/installation/daemonset/#generating-a-manifest).
```bash

kubectl apply -f https://kube-vip.io/manifests/rbac.yaml

#export KVVERSION=$(curl -sL https://api.github.com/repos/kube-vip/kube-vip/releases | jq -r ".[0].name") #if needed


alias kube-vip="KVVERSION=$(curl -sL https://api.github.com/repos/kube-vip/kube-vip/releases | jq -r ".[0].name"); ctr image pull ghcr.io/kube-vip/kube-vip:$KVVERSION; ctr run --rm --net-host ghcr.io/kube-vip/kube-vip:$KVVERSION vip /kube-vip"

alias kube-vip="ctr image pull ghcr.io/kube-vip/kube-vip:latest; ctr run --rm --net-host ghcr.io/kube-vip/kube-vip:latest vip /kube-vip"
kube-vip manifest daemonset --interface ens160 --address <VIRTUAL_IP_1> --inCluster --taint --controlplane --arp --leaderElection | tee /var/lib/rancher/k3s/server/manifests/kube-vip.yaml
```

# 4. Deploy and Configure MetalLB
> You can find more information about **MetalLB**s [installation](https://metallb.universe.tf/installation/).
> You can find more information about **MetalLB**s [configuration](https://metallb.universe.tf/configuration/#layer-2-configuration).
```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.10/config/manifests/metallb-native.yaml
```

```bash
vim metallb-ipAddressPools.yml
```
```bash
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - <VIRTUAL_IP_2>-<VIRTUAL_IP_2>   # change this to your IP range
```
```bash
kubectl apply -f metallb-ipAddressPools.yml
```
 ```bash
vim metallb-l2advertisement.yaml
```
```bash
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: example
  namespace: metallb-system
spec:
  ipAddressPools:
  - first-pool
```
```bash
kubectl apply -f metallb-l2advertisement.yaml
```
```bash
kubectl describe service -n kube-system traefik   # find ip address that metallb gave it to service
```
> **Note:**  VIRTUAL_IP_2 is the another virtual IP.
# 5. Enable Traefik dashbourd

```bash
vim /var/lib/rancher/k3s/server/manifests/traefik-config.yaml
```
```bash
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: traefik
  namespace: kube-system
spec:
  valuesContent: |-
    additionalArguments:
      - "--api"
      - "--api.dashboard=true"
      - "--api.insecure=true"
      - "--log.level=DEBUG"
    ports:
      traefik:
        expose: true
    providers:
      kubernetesCRD:
        allowCrossNamespace: true
```
```bash
vim traefik-dashbourd-ing.yaml
```
```bash
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: traefik-dashboard
  namespace: kube-system   # Change this to the appropriate namespace if necessary
spec:
  rules:
    - host: traefik-dashboard.example.com   # Change this to your desired hostname
      http:
        paths:
          - pathType: Prefix
            path: /
            backend:
              service:
                name: traefik
                port:
                  number: 9000
```

# 6. Install helm


```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3

chmod 700 get_helm.sh

./get_helm.sh
```

# 7. Install rancher with helm chart


```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm list -A
```
```bash
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.11.0 \
  --set installCRDs=true

kubectl apply --validate=false -f https://github.com/cert-manager/cert-manager/releases/download/v1.10.0/cert-manager.crds.yaml # if needed
```
```bash
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
#helm repo add rancher-latest https://releases.rancher.com/server-charts/latest

helm install rancher rancher-stable/rancher \
  --namespace cattle-system \
  --create-namespace \
  --set hostname=rancher.example.com \
  --set certmanager.version=v1.11.0 \
  --set replicas=1 \
  --set bootstrapPassword=sbGqkyYErDV7KY5 \
  --version 2.7.5
```

>**Error** unable to build kubernetes objects from release manifest: resource mapping not found for name: "rancher" namespace

>**Note**  If you encounter above error,  you need to _remove_ the argument v from the ( --set certmanager.version=1.11.0) or ( --set certmanager.version=1.10.0)

# 8. You can find password UI with this command
```bash
user: admin
password:
kubectl get secret --namespace cattle-system bootstrap-secret -o go-template='{{ .data.bootstrapPassword|base64decode}}{{ "\n" }}'

```


# Related

```
* https://github.com/rancher/rancher/issues/41650
* https://github.com/rancher/rancher/issues/41650
* https://rancher.com/docs/rancher/v1.6/en/installing-rancher/installing-server/no-internet-access/#using-a-private-registry
* https://github.com/rancher/rancher/issues/36354 
```
