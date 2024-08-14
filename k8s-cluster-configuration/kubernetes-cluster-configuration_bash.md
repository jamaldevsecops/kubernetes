Here is a Bash script that automates the configuration of a Kubernetes cluster on a master node using `kubeadm` with containerd as the container runtime. This script assumes the server is running Ubuntu 20.04 LTS.
```sh
#!/bin/bash

# 1. Set hostname to 'controlplane'
sudo hostnamectl set-hostname controlplane
echo "Hostname set to controlplane"

# 2. Update /etc/hosts
grep -qxF '127.0.1.1 controlplane' /etc/hosts || echo '127.0.1.1 controlplane' | sudo tee -a /etc/hosts
echo "Updated /etc/hosts"

# 3. Comment swap entries in /etc/fstab
sudo sed -i '/\/dev\/disk\/by-id\/dm-uuid-LVM/d' /etc/fstab
sudo sed -i '/\/swap.img/d' /etc/fstab
echo "Swap entries commented in /etc/fstab"

# 4. Disable swap
sudo swapoff -a
sudo swapon --show
echo "Swap disabled"

# 5. Load necessary modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter
echo "Necessary kernel modules loaded"

# 6. Configure sysctl
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system
echo "Sysctl parameters set"

# 7. Verify kernel modules and sysctl settings
lsmod | grep br_netfilter && lsmod | grep overlay
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
echo "Kernel modules and sysctl settings verified"

# 8. Install containerd
curl -fsSL https://get.docker.com -o install-docker.sh
sudo bash install-docker.sh
sudo mv /etc/containerd/config.toml /etc/containerd/config.toml.backup
sudo systemctl restart containerd
echo "Containerd installed and configured"

# 9. Install Kubernetes packages
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
sudo mkdir -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.27/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.27/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
echo "Kubernetes packages installed and held"

# 10. Initialize the Kubernetes cluster
sudo kubeadm init
echo "Kubernetes cluster initialized"

# 11. Configure kubectl for regular user
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
echo "kubectl configured for regular user"

# 12. Verify system pods and nodes
kubectl get pods -n kube-system
kubectl get nodes -o wide

# 13. Install Calico network plugin
curl https://raw.githubusercontent.com/projectcalico/calico/v3.24.1/manifests/calico.yaml -O
kubectl apply -f calico.yaml
echo "Calico network plugin installed"

# 14. Wait for Calico to initialize
echo "Waiting for Calico to initialize..."
while ! kubectl get nodes | grep -q "Ready"; do
  echo "Waiting for nodes to be ready..."
  sleep 5
done

# 15. Final verification of nodes and system pods
kubectl get nodes
kubectl get pods -n kube-system
