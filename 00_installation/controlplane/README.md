### Prepare system
```bash
sudo apt update
sudo apt install -y apt-transport-https ca-certificates curl

# disable swap
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Load necessary kernel modules
sudo modprobe overlay
echo "br_netfilter" | sudo tee /etc/modules-load.d/k8s.conf
sudo modprobe br_netfilter

sudo tee /etc/sysctl.d/kubernetes.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
sudo sysctl --system
```
<details>
  <summary>Details</summary>

  ### Disable Swap
  1. Reason: Kubernetes requires swap to be disabled to ensure optimal scheduling and performance.
  2. Why? 
  - The Kubernetes scheduler assumes that nodes have consistent memory availability. If swap is enabled, the OS might swap out critical Kubernetes processes, leading to unpredictable behavior, degraded performance, and potential failures.
  3. Kubelet Enforces This:
  - Since Kubernetes v1.22, kubeadm checks if swap is enabled and refuses to proceed if it is.
  
  ### Load Necessary Kernel Modules
  1. Modules Loaded:
  - overlay → Improves filesystem performance, especially with Containerd and overlay2 storage driver.
  - br_netfilter → Enables iptables rules to be applied to bridged network traffic, which is crucial for Kubernetes networking.
  2. Why?
  - Kubernetes uses iptables for routing network packets between pods and services.
  - The br_netfilter module ensures that packets traversing a Linux bridge (used in container networking) are correctly processed by iptables.
  - Without this, inter-pod communication may not work as expected.

  ### Set Up Sysctl Parameters
  1. Why?
  - net.bridge.bridge-nf-call-iptables = 1: Ensures that bridged network traffic goes through iptables for proper filtering and NAT handling.
  - net.bridge.bridge-nf-call-ip6tables = 1: Same as above, but for IPv6.
  - net.ipv4.ip_forward = 1: Enables packet forwarding, allowing the node to route traffic between pods and external networks.
  2. Without these settings:
  - Pod-to-pod and pod-to-service communication may fail.
  - Cluster networking may break, causing issues with CNI plugins like Flannel or Calico.

  ### SUMMARY
  1. Kubernetes will refuse to start (swap issue).
  2. Containers may not be able to communicate (iptables/ip_forward issue).
  3. Networking between pods and services may break (br_netfilter issue).
</details>

### Install containerd
```bash
  sudo apt install -y containerd

  sudo mkdir -p /etc/containerd
  sudo containerd config default | sudo tee /etc/containerd/config.toml

  sudo systemctl restart containerd
  sudo systemctl enable containerd

```

### Install Kubernetes Components (v1.32)
```bash
mkdir -p /etc/apt/keyrings/
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### Initialize the Kubernetes Control Plane
```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# install flannel
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

### Ports
- 6443: api server
- 10250: kubelet
- 2379-2381: etcd
- udp:8472: flannel VXLAN traffic
