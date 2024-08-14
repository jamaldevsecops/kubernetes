Here is a Bash script that automates the configuration of a Kubernetes cluster on a worker node using kubeadm with containerd as the container runtime. This script assumes the server is running Ubuntu 20.04 LTS.
```sh
#!/bin/bash

# Prompt for the hostname
read -p "Enter the desired hostname: " HOSTNAME

# 1. Set the provided hostname
sudo hostnamectl set-hostname $HOSTNAME
echo "Hostname set to $HOSTNAME"

# 2. Update /etc/hosts
grep -qxF "127.0.1.1 $HOSTNAME" /etc/hosts || echo "127.0.1.1 $HOSTNAME" | sudo tee -a /etc/hosts
echo "/etc/hosts updated with $HOSTNAME"

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
sudo apt-get install -y kubelet kubeadm
sudo apt-mark hold kubelet kubeadm
echo "Kubernetes packages installed and held"

# 10. Copy the generated token and apply here
echo "Copy the generated token on the master node and past here to be added on the cluster"
```
