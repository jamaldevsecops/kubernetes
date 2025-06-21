## Remove Worker Nodes from the Cluster

# Step 1: Drain and Remove Nodes from the Cluster
# Run this from the control-plane node
```
kubectl drain crm-dev.apsis.localnet --ignore-daemonsets --delete-emptydir-data
kubectl drain erp-dev.apsis.localnet --ignore-daemonsets --delete-emptydir-data
```
```
kubectl delete node crm-dev.apsis.localnet
kubectl delete node erp-dev.apsis.localnet
```
# Step 2: Reset Kubernetes on the Target Nodes
# Run this on both crm-dev and erp-dev nodes
```
sudo kubeadm reset -f
sudo rm -rf /etc/kubernetes /var/lib/etcd /var/lib/kubelet /var/lib/cni /var/run/kubernetes
```
# Step 3: Uninstall Kubernetes Components
# Run this on both crm-dev and erp-dev nodes
```
sudo apt-get purge -y kubeadm kubelet kubectl
sudo apt-get autoremove -y
sudo rm -rf ~/.kube
```
# Step 4: Verify Cluster Status
# Run this on the control-plane node to check if the nodes are removed
```
kubectl get nodes
```
