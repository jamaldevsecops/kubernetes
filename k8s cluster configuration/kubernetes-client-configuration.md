# Configurering Kubernetes Client

System Requirements:  
OS: Ubuntu 20.04 LTS;  
Core: 2;  
Memory: 4 GB;  
Disk: 70 GB

On Worker Node, Run the Following Commands:
-------------------------------------------
-------------------------------------------
Here all configuration has been performed for one node (worker1), if you want to add additional node(s), you have to perform the same. 

Change server's hostname to 'worker1'
```
sudo hostnamectl set-hostname worker1
bash
```

Make sure the following line exists in /etc/hosts file.
```
127.0.1.1 worker1
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
Note: In releases older than Debian 12 and Ubuntu 22.04, /etc/apt/keyrings does not exist by default; you can create it by running sudo mkdir -m 755 /etc/apt/keyrings
```
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.27/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
sudo echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.27/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
```

Install kubelet, kubeadm, and kubectl
```
sudo apt-get install -y kubelet kubeadm
```

Hold the versions to prevent upgrades
```
sudo apt-mark hold kubelet kubeadm
```

Check kubeadm version
```
kubeadm version
```


# Add worker node(s) to the cluster 
### On Master node, generate token
```
kubeadm token list
kubeadm token create --print-join-command
```
Sample Output: 
```
root@master1:~# kubeadm token create --print-join-command
kubeadm join 192.168.20.126:6443 --token ixvylp.epn8wky4simkv5qx --discovery-token-ca-cert-hash sha256:3b88ccc1366ae5e0239202b56dbdcca27948fa9cfa65d3218cb50d503f65fb23 
root@master1:~# 
```
### On Worker node, execute the output. (If Docker Engine as Container Runtime)
Note: Pass the domain socket ```unix:///var/run/cri-dockerd.sock``` for Docker Engine with the generated token. 
```
kubeadm join 192.168.20.126:6443 --token ixvylp.epn8wky4simkv5qx --discovery-token-ca-cert-hash sha256:3b88ccc1366ae5e0239202b56dbdcca27948fa9cfa65d3218cb50d503f65fb23 --cri-socket unix:///var/run/cri-dockerd.sock
```
### Verify from the Master node. 
```
kubectl get nodes -o wide
```
Sample Output: 
```
root@master1:~# kubectl get nodes -o wide
NAME      STATUS   ROLES           AGE   VERSION    INTERNAL-IP      EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
master1   Ready    control-plane   73m   v1.27.14   192.168.20.126   <none>        Ubuntu 20.04.2 LTS   5.4.0-137-generic   docker://26.1.3
worker1   Ready    <none>          10m   v1.27.14   192.168.20.127   <none>        Ubuntu 20.04.6 LTS   5.4.0-81-generic    docker://26.1.3
worker2   Ready    <none>          54s   v1.27.14   192.168.20.128   <none>        Ubuntu 20.04.6 LTS   5.4.0-81-generic    docker://26.1.3
root@master1:~#
```
### On Worker node, execute the output. (If containerd as Container Runtime)
```
kubeadm join 192.168.20.126:6443 --token ixvylp.epn8wky4simkv5qx --discovery-token-ca-cert-hash sha256:3b88ccc1366ae5e0239202b56dbdcca27948fa9cfa65d3218cb50d503f65fb23
```
### Verify from the Master node. 
```
kubectl get nodes -o wide
```


