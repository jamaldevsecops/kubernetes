Here is a Bash script that automates the configuration of a Kubernetes cluster on a worker node using kubeadm with containerd as the container runtime. This script assumes the server is running Ubuntu 20.04 LTS.
```sh
#!/bin/bash

# Kubernetes Worker Node Setup Script (v1.33.1, containerd)
# Run as root or with sudo on Ubuntu 20.04/22.04
# Config: Kubernetes v1.33.1, joins master at explicit control plane IP

set -e
trap 'echo "❌ Error occurred at line $LINENO. Exiting..."' ERR

# Variables
K8S_VERSION="1.33.1"
NODE_NAME="k8s-worker1.apsis.localnet"
CONTROL_PLANE_IP="192.168.20.126"

echo "Starting Kubernetes Worker Node Setup..."
echo "Worker Node: $NODE_NAME"
echo "Control Plane IP: $CONTROL_PLANE_IP"
echo "Kubernetes Version: $K8S_VERSION"

# Step 1: Update Hostname
echo "Setting hostname to $NODE_NAME..."
hostnamectl set-hostname $NODE_NAME
# Replace or add 127.0.1.1 entry in /etc/hosts
if grep -q "^127\.0\.1\.1" /etc/hosts; then
  sed -i "s/^127\.0\.1\.1.*/127.0.1.1 $NODE_NAME/" /etc/hosts
else
  echo "127.0.1.1 $NODE_NAME" >> /etc/hosts
fi
echo "✅ Hostname set"

# Step 2: Prepare the System
echo "Updating system..."
apt update && apt upgrade -y
echo "✅ System updated"

echo "Disabling swap if enabled..."
if grep -q '/dev/disk/by-id/dm-uuid-LVM' /etc/fstab; then
  sed -i '/\/dev\/disk\/by-id\/dm-uuid-LVM/d' /etc/fstab
  echo "Removed LVM swap entry from /etc/fstab"
fi
if grep -q '/swap.img' /etc/fstab; then
  sed -i '/\/swap.img/d' /etc/fstab
  echo "Removed /swap.img entry from /etc/fstab"
fi
if swapon --summary | grep -q '^'; then
  swapoff -a
  echo "Swap turned off"
else
  echo "Swap already off"
fi
swapon --show
echo "✅ Swap disabled"

echo "Loading kernel modules..."
modprobe overlay
modprobe br_netfilter
cat <<EOF >/etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
echo "✅ Kernel modules loaded"

echo "Configuring sysctl parameters..."
cat <<EOF >/etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
sysctl --system
echo "✅ Sysctl parameters configured"

echo "Installing required tools..."
apt install -y curl apt-transport-https ca-certificates gnupg
echo "✅ Required tools installed"

# Step 3: Install containerd
echo "Installing containerd..."
apt install -y containerd
echo "✅ containerd installed"

echo "Configuring containerd..."
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
echo "✅ containerd configured"

echo "Restarting containerd..."
systemctl restart containerd
systemctl enable containerd
echo "✅ containerd restarted"

echo "Verifying containerd..."
ctr version || { echo "❌ containerd verification failed"; exit 1; }
echo "✅ containerd verified"

# Step 4: Install Kubernetes Components
echo "Adding Kubernetes APT repository..."
[ -d /etc/apt/keyrings ] || sudo mkdir -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /" > /etc/apt/sources.list.d/kubernetes.list
echo "✅ Kubernetes repository added"

echo "Installing kubeadm, kubelet, kubectl..."
apt update
apt install -y kubeadm=$K8S_VERSION-* kubelet=$K8S_VERSION-* kubectl=$K8S_VERSION-*
apt-mark hold kubeadm kubelet kubectl
echo "✅ Kubernetes components installed"

echo "Installing cri-tools..."
apt install -y cri-tools
echo "✅ cri-tools installed"

echo "Verifying kubeadm version..."
kubeadm version || { echo "❌ kubeadm verification failed"; exit 1; }
echo "✅ kubeadm verified"

# Step 5: Join the Cluster
echo "Please provide the kubeadm join command from the master node."
echo "Example: kubeadm join $CONTROL_PLANE_IP:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>"
echo "Run the command with --cri-socket unix:///var/run/containerd/containerd.sock"
echo "Example:"
echo "sudo kubeadm join $CONTROL_PLANE_IP:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash> --cri-socket unix:///var/run/containerd/containerd.sock"
echo "✅ Setup complete, waiting for manual join command execution"

echo "✅ Kubernetes Worker Node Setup Complete!"
```
