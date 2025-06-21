# Kubernetes Master Node Setup with containerd (Kubernetes v1.33.1)

This guide provides detailed steps to install a single Kubernetes master node using containerd as the container runtime on Ubuntu 20.04/22.04. The setup uses Kubernetes v1.33.1, containerd v1.7.x, and a pod subnet of 172.16.0.0/12.
## Prerequisites
  - System:
      - OS: Ubuntu 20.04 or 22.04.
      - CPU: 2 cores (minimum).
      - RAM: 2 GB (4 GB recommended).
      - Disk: 20 GB free.
      - Network: Static IP (`192.168.20.126` in this case) and internet access.
      - Root or sudo access.
  - Software:
      - Kubernetes: v1.33.1.
      - containerd: v1.7.x.
      - cri-tools: v1.33.0.
  - Assumptions:
      - Single master node (not high-availability).
      - Using `kubeadm` for cluster initialization.
      - Pod network CIDR: `172.16.0.0/12`.
      - CNI plugin: Calico (compatible with `172.16.0.0/12`).

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

**Step 4: Prepare kubeadm Configuration**  
1. **Create Initial Configuration File (`kubeadm-config.yaml`):**
```bash
cat <<EOF | sudo tee kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
nodeRegistration:
  criSocket: unix:///var/run/containerd/containerd.sock
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: v1.33.1
controlPlaneEndpoint: "192.168.20.128:6443"
networking:
  podSubnet: "172.16.0.0/12"
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
EOF
```
2. **Migrate Configuration to v1beta4 (since `v1beta3` is deprecated):**
```bash
sudo kubeadm config migrate --old-config kubeadm-config.yaml --new-config kubeadm-config-v1beta4.yaml
```
3. **Verify Migrated Configuration:**
```bash
cat kubeadm-config-v1beta4.yaml
```
The file should use apiVersion: kubeadm.k8s.io/v1beta4 and retain all settings, e.g.:
```yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: InitConfiguration
nodeRegistration:
  criSocket: unix:///var/run/containerd/containerd.sock
---
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
kubernetesVersion: v1.33.1
controlPlaneEndpoint: "192.168.20.128:6443"
networking:
  podSubnet: "172.16.0.0/12"
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
```

**Step 5: Initialize the Kubernetes Master Node**  
1. **Initialize the Cluster:**
```bash
sudo kubeadm init --config kubeadm-config-v1beta4.yaml
```
  - This sets up the control plane with containerd.
  - Save the kubeadm join command output for adding worker nodes later.
  - If initialization fails, use verbose logging:
  ```bash
  sudo kubeadm init --config kubeadm-config-v1beta4.yaml --v=5
  ```
2. **Set Up kubeconfig for Admin Access:**
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
3. **Verify Cluster Status:**
```bash
kubectl get nodes
```
The master node should appear, likely in NotReady status until the CNI is installed.

**Step 6: Install a Pod Network (CNI)**  
The pod subnet `172.16.0.0/12` is compatible with Calico, which we’ll use for simplicity. If you prefer Flannel, you’ll need to customize its manifest.
1. **Install Calico:**
```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```
2. **Verify CNI Pods:**
```bash
kubectl get pods -n kube-system
```
Ensure Calico pods (`calico-*`) are `Running`.

3. Check Node Status:
```bash
kubectl get nodes
```
The master node should now be Ready.

**Step 7: Configure the Master Node (Optional)**  
1. **Untaint the Master Node (to allow scheduling pods on the master, useful for single-node clusters):**
```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```
2. **Verify containerd Integration:**
```bash
kubectl get pods -A -o wide
```
Inspect a pod to confirm containerd usage:
```bash
kubectl describe pod <pod-name> -n kube-system
```

**Step 8: Test the Cluster**  
1. **Deploy a Test Pod:**
```bash
kubectl run nginx --image=nginx --restart=Never
```
2. **Verify the Pod:**
```bash
kubectl get pods
```
The nginx pod should be Running.

**Step 9: Troubleshooting**  
  - **containerd Issues:**
      - Check status: sudo systemctl status containerd.
      - Verify CRI socket: sudo crictl --runtime-endpoint unix:///var/run/containerd/containerd.sock ps.
  - **kubeadm Issues:**
      - View logs: sudo journalctl -u kubelet.
      - Reset cluster: sudo kubeadm reset.
  - **Node NotReady:**
      - Ensure CNI pods are running: kubectl get pods -n kube-system.
      - Check Calico logs: kubectl logs -n kube-system <calico-pod-name>.
  - **Version Mismatch:**
      - Verify repository: cat /etc/apt/sources.list.d/kubernetes.list.
      - Reinstall kubeadm: sudo apt install -y kubeadm=1.33.1-*.

## Notes  
  - **Kubernetes Version:** Using v1.33.1 as specified. Ensure your repository supports this version.
  - **Pod Subnet:** 172.16.0.0/12 works with Calico. For Flannel, use 10.244.0.0/16 or customize the manifest.
  - **Next Steps:** Use the kubeadm join command to add worker nodes.
  - **Security:** Configure RBAC, network policies, and encryption for production use.
  - **References:**
      - Kubernetes Docs: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/
      - containerd Docs: https://containerd.io/docs/
      - Calico Docs: https://docs.projectcalico.org/
