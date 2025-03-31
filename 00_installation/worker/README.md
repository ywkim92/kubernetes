## Install nvidia, cuda and container toolkit
### nvidia
```bash
(writing)
```
### cuda
```bash
# set env variables
echo 'export PATH=/usr/local/cuda-12.6/bin:$PATH' | sudo tee -a /etc/profile.d/cuda.sh
echo 'export LD_LIBRARY_PATH=/usr/local/cuda-12.6/lib64:$LD_LIBRARY_PATH' | sudo tee -a /etc/profile.d/cuda.sh
source /etc/profile.d/cuda.sh
```
### nvidia container toolkit
```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
echo "deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://nvidia.github.io/libnvidia-container/stable/ubuntu$distribution/$(ARCH) /" | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
sudo apt update

sudo apt install -y nvidia-container-toolkit

# after containerd installed
sudo nvidia-ctk runtime configure --runtime=docker --set-as-default
sudo systemctl restart containerd
```
### test(optional)
```bash
sudo ctr image pull docker.io/nvidia/cuda:12.6.0-base-ubuntu22.04
sudo ctr run --rm --gpus 1 --runtime=nvidia docker.io/nvidia/cuda:12.6.0-base-ubuntu22.04 test nvidia-smi
```

## Prepare system
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

## Install containerd
```bash
sudo apt install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd
```

## Install k8s components
```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

# run on controlplane
kubeadm token create --print-join-command

# run on worker nodes
sudo kubeadm join <CONTROL_PLANE_IP>:6443 --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH>

# install nvidia k8s-device-plugin
kubectl create -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.17.0/deployments/static/nvidia-device-plugin.yml
```

## Verification and Test
```bash
kubectl get nodes -o jsonpath='{.items[*].status.allocatable}'

kubectl describe node <worker-node-name> | grep -i gpu

kubectl run gpu-test --rm -it --restart=Never --image=nvidia/cuda:12.6-base --limits=nvidia.com/gpu=1 -- nvidia-smi
```

## Ports
- 6443
- 10250
- 30000-32767
- 443

## kubeadm reset
on worker node only
```bash
sudo kubeadm reset -f

sudo ip link delete flannel.1
sudo ip link delete cni0
```

on controlplane too
```bash
# on controlplane
sudo kubeadm reset -f

sudo ip link delete flannel.1
sudo ip link delete cni0

sudo kubeadm init --pod-network-cidr=10.244.0.0/16
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
