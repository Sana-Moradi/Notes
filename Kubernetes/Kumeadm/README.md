# Install kubernetes (1.26.5) with kubeadm on Ubuntu (20.04)

## Prerequisites
### Product UUID
Ensure hosts have unique product_uuid by running the below command. If two nodes have the same product_uuid, the Kubernetes cluster installation will fail.
```bash
sudo cat /sys/class/dmi/id/product_uuid
```
### Set Hostname
Set a unique hostname for all your nodes. Example:
```bash
sudo hostnamectl set-hostname example.local
```
# 1. Install Keepalived on all Master Nodes
 SSH into each of your master nodes and  Install Keepalived:

   
```bash
sudo apt update
sudo apt install keepalived
```
 Add the following configuration (adjust values as needed):
 
```bash
sudo nano /etc/keepalived/keepalived.conf
```
```
vrrp_script chk_kubeapi {
    script "/usr/bin/curl -f http://localhost:6443/healthz"
    interval 2
    weight -2
}

vrrp_instance VI_1 {
    state MASTER
    interface ens160  # Replace with the appropriate network interface
    virtual_router_id 51
    priority 100  # Adjust priority on each master node (100, 101, 102)
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass YOUR_SECRET_PASSWORD
    }
    virtual_ipaddress {
        <VIRTUAL_IP>  # Virtual IP address
    }
    track_script {
        chk_kubeapi
    }
}
```
> **Note:**    Replace ens160 with the appropriate network interface on your servers and set a secure YOUR_SECRET_PASSWORD for authentication.

Save the file and exit the text editor.
Enable and start Keepalived:

```bash
 sudo systemctl enable keepalived
 sudo systemctl start keepalived
```
> **Note:** Stop keepalived in master2, master3 and start them at end of cluster is up
# 2. Install HAProxy on the Master Nodes

Repeat these steps on all three master nodes.
SSH into each of your master nodes.
Install HAProxy:
```bash
sudo apt update
sudo apt install haproxy
```
Edit the HAProxy configuration file:
```bash
sudo nano /etc/haproxy/haproxy.cfg
```
```
global
    log /dev/log local0
    log /dev/log local1 notice
    daemon

defaults
    log global
    mode tcp
    option tcplog
    option dontlognull
    timeout connect 5000ms
    timeout client 50000ms
    timeout server 50000ms

frontend kubernetes
    bind *:7443
    default_backend kubernetes-master-nodes

backend kubernetes-master-nodes
    balance roundrobin
    mode tcp
    option tcp-check
    server k8s-master-1 <MASTER_NODE01_IP>:6443 check fall 3 rise 2
    server k8s-master-2 <MASTER_NODE02_IP>:6443 check fall 3 rise 2
    server k8s-master-3 <MASTER_NODE03_IP>:6443 check fall 3 rise 2
```

Enable and start HAProxy:
```bash
sudo systemctl enable haproxy
sudo systemctl start haproxy
```

You may need to allow the below ports on the iptables for Kubernetes to work correctly.
**On Master Node**
```bash
sudo ufw allow 6443/tcp
ufw allow 2379/tcp
sudo ufw allow 2380/tcp
sudo ufw allow 10250/tcp
sudo ufw allow 10257/tcp
sudo ufw allow 10259/tcp
sudo ufw allow 7443/tcp
sudo ufw reload
```
**On Worker Nodes**
```bash
sudo ufw allow 10250/tcp
sudo ufw allow 30000:32767/tcp
sudo ufw reload
```
Ensure the master and worker nodes can resolve the hostname for all nodes in the cluster.
```bash
sudo nano /etc/hosts
```
```bash
<MASTER_NODE01_IP>  master1
<MASTER_NODE02_IP>  master2
<MASTER_NODE03_IP>  master3
<VIRTUAL_IP> vrrp-kube-ip   
```
Repeat these steps on all three master nodes.
# 3. Install kubelet, kubeadm, kubectl
Repeat these steps on all three **Master** nodes  and all three **Worker** nodes:

```bash
sudo apt -y install curl apt-transport-https
curl  -fsSL  https://packages.cloud.google.com/apt/doc/apt-key.gpg|sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/kubernetes.gpg
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt -y install vim git curl wget kubelet=1.26.5-00 kubeadm=1.26.5-00 kubectl=1.26.5-00
sudo apt-mark hold kubelet kubeadm kubectl
```

# 4. Disable Swap
Turn off swap:
```bash
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```
> **Note:** Now disable Linux swap space permanently in /etc/fstab. Search for a swap line and add # (hashtag) sign in front of the line.
```bash
sudo swapoff -a
sudo mount -a
free -h
```
# 5.  Load Modules for Bridged Traffic
Load the `br_netfilter` module to enable traffic communication between Kubernetes pods across the cluster nodes.
```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF
```
Add below entries to let iptables see the bridged traffic.
```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
```

```bash
sudo tee /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF
```

Ensure sysctl params are set:
```bash
sudo tee /etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
```
 Run the below command to apply the setting without rebooting the system.
```bash
sudo sysctl --system
sudo sysctl -p
```

# 6. Add Docker repo
Repeat these steps on all three **Master** nodes  and all three **Worker** nodes.

Install required packages:
```bash
sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates
```
Add Docker repo:
```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/docker-archive-keyring.gpg
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
```
Install containerd:
```bash
sudo apt update
sudo apt install -y containerd.io
```
Configure containerd and start service:
```bash
sudo mkdir -p /etc/containerd
sudo containerd config default|sudo tee /etc/containerd/config.toml
```
Restart containerd:
```bash
sudo systemctl restart containerd
sudo systemctl enable containerd
systemctl status containerd
```

# 7. kubeadm pull images and init cluster

Repeat these steps on all three **Master** nodes:
```bash
kubeadm config images pull --v=5 
```
Repeat these steps on  **Master1** node:
```bash
kubeadm init --control-plane-endpoint=vrrp-kube-ip:7443 --upload-certs --v=5 
```
Repeat these steps on all three **Master** nodes:
```bash
mkdir -p $HOME/.kube
sudo cp -f /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
# 8. Install Calico
Install Calico to provide both networking and network policy for self-managed on-premises deployments.

```bash
curl https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/calico.yaml -O
kubectl apply -f calico.yaml
```

# 9. Get join Token
A token is required when joining a new worker node to the Kubernetes cluster.
SSH into your **Master1** nodes:
```bash
kubeadm token create --print-join-command    
```
**Command output:**
```bash
Then you can join any number of worker nodes by running the following on each as root:

kubeadm join vrrp-kube-ip:7443 --token ex7703.sqnravr7pudyyez4 \
        --discovery-token-ca-cert-hash sha256:68e26859ddf2a3086142d952ef4ac901440d6fa0f72dd36bf920ac2a2710ecd8


You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join vrrp-kube-ip:7443 --token ex7703.sqnravr7pudyyez4 \
        --discovery-token-ca-cert-hash sha256:68e26859ddf2a3086142d952ef4ac901440d6fa0f72dd36bf920ac2a2710ecd8 \
        --control-plane --certificate-key d603babe514cf19b516568f64200102948aa08e78ed0aa836447a21c1fbb783c
        --ignore-preflight-errors=all
```
