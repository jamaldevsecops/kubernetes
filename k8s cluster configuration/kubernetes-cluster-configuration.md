# Configurering Kubernetes Cluster with Kubeadm

System Requirements:  
OS: Ubuntu 20.04 LTS;  
Core: 2;  
Memory: 4 GB;  
Disk: 70 GB

On Master Node, Run the Following Commands:
-------------------------------------------
-------------------------------------------
Change server's hostname to 'master1'
```
sudo hostnamectl set-hostname master1
bash
```

Make sure the following line exists in /etc/hosts file.
```
127.0.1.1 master1
```

Make sure the following lines are commented in /etc/fstab file.
```
/dev/disk/by-id/dm-uuid-LVM-3FUB4eGR4Sih9Dg3qmmXaWxQJcyGufjiqHJ8AOoU2U0FOPBXSp558oqhW69MEXVq none swap sw 0 0
/swap.img none swap sw 0 0
```

Disable swap to prevent Kubernetes issues
```
sudo swapoff -a
sudo swapon --show
```

Forwarding IPv4 and letting iptables see bridged traffic:  
Ref: https://v1-27.docs.kubernetes.io/docs/setup/production-environment/container-runtimes/#docker
```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```
```
sudo modprobe overlay
sudo modprobe br_netfilter
```
sysctl params required by setup, params persist across reboots
```
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
```
Apply sysctl params without reboot
```
sudo sysctl --system
```
Verify that the br_netfilter, overlay modules are loaded by running the following commands:
```
lsmod | grep br_netfilter
lsmod | grep overlay
```
Verify that the net.bridge.bridge-nf-call-iptables, net.bridge.bridge-nf-call-ip6tables, and net.ipv4.ip_forward system variables are set to 1 in your sysctl config by running the following command:
```
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
```


Metho1: Docker Engine as a Container Runtimes
---------------------------------------
Ref: https://v1-27.docs.kubernetes.io/docs/setup/production-environment/container-runtimes/#docker
```
curl -fsSL https://get.docker.com -o install-docker.sh
sudo bash install-docker.sh
```
Install ```cri-dockerd``` adapter to integrate Docker Engine with Kubernetes  
Ref: https://github.com/Mirantis/cri-dockerd/releases
```
wget https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.14/cri-dockerd_0.3.14.3-0.ubuntu-focal_amd64.deb
sudo dpkg -i cri-dockerd_0.3.14.3-0.ubuntu-focal_amd64.deb
```

Method2: containerd as a Container Runtimes
-------------------------------------------
Ref: https://v1-27.docs.kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd  
Backup and restart containerd configuration
```
curl -fsSL https://get.docker.com -o install-docker.sh
sudo bash install-docker.sh
```
```
sudo mv /etc/containerd/config.toml /etc/containerd/config.toml.backup
sudo systemctl restart containerd
````




Install Kubernetes packages and setup repository
------------------------------------------------
Ref: https://v1-27.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/  
Note: In releases older than Debian 12 and Ubuntu 22.04, ```/etc/apt/keyrings``` does not exist by default; you can create it by running ```sudo mkdir -m 755 /etc/apt/keyrings```
```
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
```
```
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.27/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
sudo echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.27/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
```

Install kubelet, kubeadm, and kubectl
```
sudo apt-get install -y kubelet kubeadm kubectl
```

Hold the versions to prevent upgrades
```
sudo apt-mark hold kubelet kubeadm kubectl
```

Check kubeadm version
```
kubeadm version
```


Initialize the Kubernetes cluster (If Method1 is Used)
-------------------------------------------------------------------------------
Create a file named kubeadm-config.yaml:
```
nano kubeadm-config.yaml
```
Add the following content to specify the Docker CRI socket:
```
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: "0.0.0.0"
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///var/run/cri-dockerd.sock
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
networking:
  podSubnet: "10.244.0.0/16"
```
Run the kubeadm init command with the configuration file:
```
sudo kubeadm init --config kubeadm-config.yaml
```

Initialize the Kubernetes cluster (If Method2 is Used)
----------------------------------------------------------------------------
```
sudo kubeadm init
```

To start using your cluster, run the following as a regular user:
----------------------------------------------------------------
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Alternatively, if you are the root user, you can run:
```
export KUBECONFIG=/etc/kubernetes/admin.conf
```

Verify the status of the system pods and nodes
```
kubectl get pods -n kube-system
kubectl get nodes -o wide
```
Sample Output  
```
root@master1:~# kubectl get pods -n kube-system
NAME                              READY   STATUS    RESTARTS   AGE
coredns-5d78c9869d-75cd5          0/1     Pending   0          2m11s
coredns-5d78c9869d-gxpj7          0/1     Pending   0          2m11s
etcd-master1                      1/1     Running   0          2m27s
kube-apiserver-master1            1/1     Running   0          2m27s
kube-controller-manager-master1   1/1     Running   0          2m27s
kube-proxy-fnxmr                  1/1     Running   0          2m11s
kube-scheduler-master1            1/1     Running   0          2m29s
root@master1:~# 
root@master1:~# kubectl get nodes -o wide
NAME      STATUS     ROLES           AGE     VERSION    INTERNAL-IP      EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
master1   NotReady   control-plane   2m41s   v1.27.14   192.168.20.126   <none>        Ubuntu 20.04.2 LTS   5.4.0-137-generic   docker://26.1.3
root@master1:~# 
```
Install Calico network plugin for the cluster
```
curl https://raw.githubusercontent.com/projectcalico/calico/v3.24.1/manifests/calico.yaml -O
kubectl apply -f calico.yaml
```

Wait until you get a success message from the following command
```
tail -f /var/log/syslog
```

Verify the status of the nodes and system pods
```
kubectl get nodes
kubectl get pods -n kube-system
```
