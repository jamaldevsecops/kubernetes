We are going to upgrade Kubernetes Version 1.27 to 1.28.

Worker Node Upgradation
-----------------------
### On Master Node
```sh
sudo kubeadm upgrade plan
kubectl get nodes -o wide
kubectl cordon <worker1>
kubectl drain <worker1> --ignore-daemonsets --delete-emptydir-data --force
kubectl get nodes -o wide
```
Note: change the node name according to yours, mine is worker1

### On Worker Node
```sh
sudo apt-mark unhold kubelet kubeadm
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm
sudo apt-mark hold kubelet kubeadm
```
```sh
kubeadm upgrade node
systemctl restart kubelet
```

### On Master Node
```sh
kubectl get nodes -o wide
kubectl uncordon <worker1>
kubectl get nodes -o wide
```

Master Node Upgradation
-----------------------
### On Master Node
```sh
sudo kubeadm upgrade plan
kubectl get nodes -o wide
kubectl get pods -n kube-system
kubectl cordon <master1>
kubectl drain <master1> --ignore-daemonsets --delete-emptydir-data --force
kubectl get pods -n kube-system
```
```sh
sudo apt-mark unhold kubelet kubeadm kubectl
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl 
sudo apt-mark hold kubelet kubeadm kubectl 
```
```sh
kubeadm upgrade node
systemctl restart kubelet
```
```sh
kubectl get nodes -o wide
kubectl uncordon <master1>
kubectl get pods -n kube-system
kubectl get nodes -o wide
```


