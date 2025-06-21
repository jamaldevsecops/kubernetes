Here is a Bash script that automates the configuration of a Kubernetes cluster on a master node using `kubeadm` with containerd as the container runtime. This script assumes the server is running Ubuntu 20.04 LTS.
```sh
#!/bin/bash

# Kubernetes Master Node Setup Script (v1.33.1, containerd)
# Run as root or with sudo on Ubuntu 20.04/22.04
# Config: Kubernetes v1.33.1, pod subnet 10.244.0.0/16, control plane dynamic IP

set -e
trap 'echo "❌ Error occurred at line $LINENO. Exiting..."' ERR

# Variables
K8S_VERSION="1.33.1"
POD_SUBNET="10.244.0.0/16"
NODE_NAME="k8s-master1.apsis.localnet"
CONTROL_PLANE_IP=$(hostname -I | grep -o "192\.168\.20\.[0-9]\{1,3\}" | head -1)

if [ -z "$CONTROL_PLANE_IP" ]; then
  echo "❌ No IP address found matching 192.168.20.x"
  exit 1
fi

echo "Starting Kubernetes Master Node Setup..."
echo "Control Plane IP: $CONTROL_PLANE_IP"
echo "Pod Subnet: $POD_SUBNET"
echo "Kubernetes Version: $K8S_VERSION"

# Step 1: Update Hostname
echo "Setting hostname to $NODE_NAME..."
hostnamectl set-hostname $NODE_NAME
grep -qxF "127.0.1.1 $NODE_NAME" /etc/hosts || echo "127.0.1.1 $NODE_NAME" >> /etc/hosts
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

# Step 5: Prepare kubeadm Configuration
echo "Creating kubeadm configuration..."
cat <<EOF >/root/kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
nodeRegistration:
  criSocket: unix:///var/run/containerd/containerd.sock
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: v$K8S_VERSION
controlPlaneEndpoint: "$CONTROL_PLANE_IP:6443"
networking:
  podSubnet: "$POD_SUBNET"
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
EOF
echo "✅ kubeadm configuration created"

echo "Migrating kubeadm configuration to v1beta4..."
kubeadm config migrate --old-config /root/kubeadm-config.yaml --new-config /root/kubeadm-config-v1beta4.yaml || { echo "❌ Config migration failed"; exit 1; }
echo "✅ kubeadm configuration migrated"

# Step 6: Initialize the Cluster
echo "Initializing Kubernetes cluster..."
kubeadm init --config /root/kubeadm-config-v1beta4.yaml || { echo "❌ Cluster initialization failed"; exit 1; }
echo "✅ Cluster initialized"

echo "Setting up kubeconfig..."
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
echo "✅ kubeconfig set up"

# Step 7: Install Flannel CNI
echo "Installing Flannel CNI..."
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml || { echo "❌ Flannel installation failed"; exit 1; }
echo "✅ Flannel CNI installed"

# Step 8: Verify Cluster
echo "Waiting for node to be Ready (up to 2 minutes)..."
sleep 10
timeout 120 kubectl get nodes | grep Ready || { echo "❌ Node not Ready"; exit 1; }
echo "✅ Node Ready"

echo "Verifying pods..."
kubectl get pods -n kube-system || { echo "❌ Pod verification failed"; exit 1; }
echo "✅ Pods verified"

# Step 9: Print Join Tokens
echo "Generating join tokens for worker nodes..."
for i in 1 2 3; do
  echo "Token for worker node $i:"
  kubeadm token create --print-join-command
  echo "✅ Token $i generated"
done

echo "✅ Kubernetes Master Node Setup Complete!"
```
