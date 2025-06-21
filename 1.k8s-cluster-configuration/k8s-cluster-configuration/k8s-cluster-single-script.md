```sh
#!/bin/bash

SSH_USER="ssh_user"
SSH_PASS="ssh_password"
SSH_PORT="ssh_port"

# Check if sshpass is installed
if ! command -v sshpass &> /dev/null; then
    echo "sshpass could not be found. Please install sshpass using 'sudo apt-get install sshpass' and try again."
    exit 1
fi

# Function to run commands on remote server
run_remote() {
    local SERVER_IP="$1"
    local CMD="$2"
    sshpass -p "$SSH_PASS" ssh -o StrictHostKeyChecking=no -p "$SSH_PORT" "$SSH_USER@$SERVER_IP" "$CMD"
}

# Function to configure common settings for master and slave nodes
configure_node() {
    local SERVER_IP="$1"
    local HOSTNAME="$2"
    run_remote "$SERVER_IP" "
        sudo hostnamectl set-hostname $HOSTNAME
        grep -qxF '127.0.1.1 $HOSTNAME' /etc/hosts || echo '127.0.1.1 $HOSTNAME' | sudo tee -a /etc/hosts
        sudo sed -i '/\/dev\/disk\/by-id\/dm-uuid-LVM/d' /etc/fstab
        sudo sed -i '/\/swap.img/d' /etc/fstab
        sudo swapoff -a
        sudo swapon --show
        cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
        sudo modprobe overlay
        sudo modprobe br_netfilter
        cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
        sudo sysctl --system
        curl -fsSL https://get.docker.com -o install-docker.sh
        sudo bash install-docker.sh
        sudo mv /etc/containerd/config.toml /etc/containerd/config.toml.backup
        sudo systemctl restart containerd
        sudo apt-get update
        sudo apt-get install -y apt-transport-https ca-certificates curl
        sudo mkdir -m 755 /etc/apt/keyrings
        sudo curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
        sudo echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
        sudo apt-get update
        sudo apt-get install -y kubelet kubeadm
        sudo apt-mark hold kubelet kubeadm
    "
}

# Function to configure master server
configure_master() {
    local MASTER_IP="$1"
    local MASTER_HOSTNAME="$2"
    configure_node "$MASTER_IP" "$MASTER_HOSTNAME"
    run_remote "$MASTER_IP" "
        sudo kubeadm init
        mkdir -p \$HOME/.kube
        sudo cp -i /etc/kubernetes/admin.conf \$HOME/.kube/config
        sudo chown \$(id -u):\$(id -g) \$HOME/.kube/config
        curl https://raw.githubusercontent.com/projectcalico/calico/v3.24.1/manifests/calico.yaml -O
        kubectl apply -f calico.yaml
        echo 'Waiting for Calico to initialize...'
        while ! kubectl get nodes | grep -q 'Ready'; do
          echo 'Waiting for nodes to be ready...'
          sleep 15
        done
    "
}

# Function to configure slave server
configure_slave() {
    local SLAVE_IP="$1"
    local SLAVE_HOSTNAME="$2"
    local TOKEN="$3"
    configure_node "$SLAVE_IP" "$SLAVE_HOSTNAME"
    run_remote "$SLAVE_IP" "
        sudo $TOKEN
    "
}

# Main script logic
read -p "Enter the master server IP: " MASTER_IP
read -p "Enter the master server hostname: " MASTER_HOSTNAME

read -p "Enter the number of slave servers: " SLAVE_COUNT

SLAVE_IPS=()
SLAVE_HOSTNAMES=()
for ((i=1; i<=SLAVE_COUNT; i++)); do
    read -p "Enter the IP address of slave server $i: " SLAVE_IP
    read -p "Enter the hostname of slave server $i: " SLAVE_HOSTNAME
    SLAVE_IPS+=("$SLAVE_IP")
    SLAVE_HOSTNAMES+=("$SLAVE_HOSTNAME")
done

# Configure the master server
echo "Configuring the master server..."
configure_master "$MASTER_IP" "$MASTER_HOSTNAME"

# Wait for 2 minutes
echo "Waiting for 2 minutes before generating tokens for slave servers..."
sleep 120

# Generate tokens and configure slave servers
for ((i=0; i<SLAVE_COUNT; i++)); do
    SLAVE_IP=${SLAVE_IPS[$i]}
    SLAVE_HOSTNAME=${SLAVE_HOSTNAMES[$i]}
    TOKEN=$(run_remote "$MASTER_IP" "sudo kubeadm token create --print-join-command")
    echo "Configuring slave server at $SLAVE_IP with hostname $SLAVE_HOSTNAME using token: $TOKEN"
    configure_slave "$SLAVE_IP" "$SLAVE_HOSTNAME" "$TOKEN"
done

echo "Kubernetes cluster setup completed!"
```
