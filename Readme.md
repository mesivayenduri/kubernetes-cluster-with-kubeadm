# 🚀 Kubernetes Cluster Setup with kubeadm

![Kubernetes](https://img.shields.io/badge/Kubernetes-Cluster-blue?logo=kubernetes)
![Ubuntu](https://img.shields.io/badge/Ubuntu-20.04%2B-orange?logo=ubuntu)
![containerd](https://img.shields.io/badge/Runtime-containerd-green)

This repository contains a step-by-step guide to set up a **Kubernetes cluster** (Control Plane + Worker Nodes) using **kubeadm**, **kubelet**, and **kubectl** on **Ubuntu**.

---

## 🖥️ Prerequisites

- Ubuntu **20.04+** (tested on 22.04 as well)  
- 2+ GB RAM per node  
- 2+ vCPUs per node  
- At least **10 GB free disk space**  
- All nodes must have unique **hostnames**, **MAC addresses**, and **product UUIDs**  
- Passwordless `sudo` access  

---

## 🔧 Step 1: Enable IP Forwarding (All Nodes)

```bash
sudo nano /etc/sysctl.conf
# Add or ensure this line exists:
net.ipv4.ip_forward = 1

# Apply changes
sudo sysctl -p

sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
sudo apt install -y containerd

ps -p 1  # check if systemd is the init system

sudo mkdir -p /etc/containerd
containerd config default | sed 's/SystemdCgroup = false/SystemdCgroup = true/' \
  | sudo tee /etc/containerd/config.toml
sudo systemctl restart containerd


# Load br_netfilter
sudo modprobe br_netfilter

# Persist across reboots
echo "br_netfilter" | sudo tee /etc/modules-load.d/br_netfilter.conf

# Enable bridged traffic in iptables
echo "net.bridge.bridge-nf-call-iptables = 1" | sudo tee -a /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-ip6tables = 1" | sudo tee -a /etc/sysctl.conf

# Apply settings
sudo sysctl -p


# Add Kubernetes repo key
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Add repo
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
  https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Install kubelet, kubeadm, kubectl
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl

# Prevent accidental upgrades
sudo apt-mark hold kubelet kubeadm kubectl

# Enable kubelet
sudo systemctl enable --now kubelet



sudo kubeadm init \
  --apiserver-advertise-address=192.168.0.4 \
  --pod-network-cidr=10.244.0.0/16 \
  --upload-certs


mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config


kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml


kubeadm join 192.168.0.4:6443 --token <your-token> \
  --discovery-token-ca-cert-hash sha256:<your-hash>

