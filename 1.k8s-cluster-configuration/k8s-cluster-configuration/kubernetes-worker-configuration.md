# Kubernetes Worker Node Setup with containerd (Kubernetes v1.33.1)

This guide provides detailed steps to configure a Kubernetes worker node using containerd as the container runtime on Ubuntu 20.04/22.04, joining it to a Kubernetes cluster with a master node running v1.33.1. The setup assumes the master node uses a pod subnet of 172.16.0.0/12 and a control plane endpoint at 192.168.20.128:6443.

## Prerequisites
- **System:**
  - OS: Ubuntu 20.04 or 22.04.
  - CPU: 2 cores (minimum).
  - RAM: 2 GB (4 GB recommended).
  - Disk: 20 GB free.
  - Network: Static IP and internet access (ensure connectivity to master node at 192.168.20.126:6443).
  - Root or sudo access.
- **Software:**
  - Kubernetes: v1.33.1 (to match master node).
  - containerd: v1.7.x.
  - cri-tools: v1.33.0.
- **Cluster Information:**
  - `kubeadm join` command from the master node initialization, including the token and discovery-token-ca-cert-hash. Example:
  ```bash
  kubeadm join 192.168.20.128:6443 --token <token> \
      --discovery-token-ca-cert-hash sha256:<hash>
  ```
  - If you don’t have the token, generate a new one on the master node:
  ```bash
  kubeadm token create --print-join-command
  ```
- **Assumptions:**
  - The master node is already initialized and running with Calico CNI (pod subnet 172.16.0.0/12).
  - The worker node will join the existing cluster as a compute node (no control plane components).

## Step-by-Step Instructions
**Step 1: Prepare the System**  
1. **Update the System:**
```bash
sudo apt update && sudo apt upgrade -y
```
2. **Disable Swap (required by Kubernetes):**
```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```
3. **Load Kernel Modules:**
```bash
sudo modprobe overlay
sudo modprobe br_netfilter
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```
4. **Configure Sysctl Parameters:**
```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
sudo sysctl --system
```
5. **Install Required Tools:**
```bash
sudo apt install -y curl apt-transport-https ca-certificates gnupg
```

**Step 2: Install containerd**  
1. **Install containerd:**
```bash
sudo apt install -y containerd
```
2. **Configure containerd:**
    - Generate default configuration:
    ```bash
    sudo mkdir -p /etc/containerd
    containerd config default | sudo tee /etc/containerd/config.toml
    ```
    - Set systemd as the cgroup driver (required for Kubernetes):
    ```bash
    sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
    ```
3. **Restart and Enable containerd:**
```bash
sudo systemctl restart containerd
sudo systemctl enable containerd
```
4. **Verify containerd:**
```bash
sudo ctr version
```
Confirm version is 1.7.x or compatible.

**Step 3: Install Kubernetes Components**  
1. **Add Kubernetes APT Repository:**
```bash
sudo curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
sudo chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
2. **Install kubeadm, kubelet, and kubectl:**
```bash
sudo apt update
sudo apt install -y kubeadm=1.33.1-* kubelet=1.33.1-* kubectl=1.33.1-*
sudo apt-mark hold kubeadm kubelet kubectl
```
3. **Install cri-tools (for containerd CRI compatibility):**
```bash
sudo apt install -y cri-tools
```
4. **Verify kubeadm Version:**
```bash
kubeadm version
```
Ensure it reports v1.33.1.
5. **List Available kubeadm Versions (optional, for reference):**
```bash
apt list -a kubeadm
```

**Step 4: Join the Kubernetes Cluster**
1. **Prepare the Join Command:**
  - Use the kubeadm join command provided during master node initialization. It should look like:
  ```bash
  kubeadm join 192.168.20.128:6443 --token <token> \
      --discovery-token-ca-cert-hash sha256:<hash>
  ```
  - If you don’t have the command, generate a new one on the master node:
  - kubeadm token create --print-join-command
2. **Execute the Join Command: Run the kubeadm join command on the worker node with sudo:**
```bash
sudo kubeadm join 192.168.20.128:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash> \
    --cri-socket unix:///var/run/containerd/containerd.sock
```
  - The --cri-socket flag ensures containerd is used as the runtime.
  - This command connects the worker node to the master node and configures it as part of the cluster.
3. **Verify Join Success: On the master node, check if the worker node has joined:**
```bash
kubectl get nodes
```
The worker node should appear, likely in NotReady status until the CNI is fully applied.

**Step 5: Verify Cluster Integration**
1. **Check Node Status (on master node):**
```bash
kubectl get nodes -o wide
```
The worker node should transition to Ready once the CNI (Calico, from master node setup) is active. This may take a few minutes.
2. **Verify Pods (on master node):**
```bash
kubectl get pods -A -o wide
```
Ensure system pods on the worker node (e.g., kube-proxy) are Running. The --o wide flag shows which node hosts each pod.
3. **Test Workload Scheduling (on master node): Deploy a test pod to confirm the worker node can run workloads:**
```bash
kubectl run nginx --image=nginx --restart=Never
```
Check if the pod is scheduled on the worker node:
```bash
kubectl get pods -o wide
```

**Step 6: Troubleshooting**
- containerd Issues:
  - Check status: sudo systemctl status containerd.
  - Verify CRI socket: sudo crictl --runtime-endpoint unix:///var/run/containerd/containerd.sock ps.
- kubeadm Join Issues:
  - View logs: sudo journalctl -u kubelet.
  - Reset and retry: sudo kubeadm reset, then re-run the join command.
  - Ensure the token is valid. If expired, generate a new one on the master node:
  ```bash
  kubeadm token create --print-join-command
  ```
- Node NotReady:
  - Confirm Calico pods are running on the master node: kubectl get pods -n kube-system.
  - Check kubelet logs on the worker node: sudo journalctl -u kubelet.
- Network Issues:
  - Verify connectivity to the master node: ping 192.168.20.128.
  - Ensure port 6443 is open: nc -zv 192.168.20.128 6443.
- Version Mismatch:
  - Verify repository: cat /etc/apt/sources.list.d/kubernetes.list.
  - Reinstall kubeadm: sudo apt install -y kubeadm=1.33.1-*.

Notes
- Kubernetes Version: The worker node uses v1.33.1 to match the master node.
- CNI: The worker node relies on the master node’s Calico CNI (pod subnet 172.16.0.0/12). No additional CNI installation is needed on the worker.
- Security: Ensure firewall rules allow traffic between master and worker nodes (ports 6443, 10250, and CNI-specific ports).
- Scaling: Repeat these steps for additional worker nodes.
- References:
  - Kubernetes Docs: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/
  - containerd Docs: https://containerd.io/docs/
  - Calico Docs: https://docs.projectcalico.org/


