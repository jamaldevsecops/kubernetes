# Static Pod: Schedule a pod on a specific node using nodeName  
Note that nodeName is used for Pods, not Deployments. If you want to use a Deployment, you should use nodeSelector, nodeAffinity, or taints and tolerations.

```
kubectl get node -o wide
```
Sample Output: 
```
NAME      STATUS   ROLES           AGE   VERSION    INTERNAL-IP      EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
master1   Ready    control-plane   31d   v1.28.10   192.168.20.126   <none>        Ubuntu 20.04.2 LTS   5.4.0-189-generic   containerd://1.6.33
worker1   Ready    <none>          31d   v1.28.10   192.168.20.127   <none>        Ubuntu 20.04.6 LTS   5.4.0-189-generic   containerd://1.6.33
worker2   Ready    <none>          31d   v1.28.10   192.168.20.128   <none>        Ubuntu 20.04.6 LTS   5.4.0-189-generic   containerd://1.6.33
worker3   Ready    <none>          24m   v1.28.11   192.168.20.129   <none>        Ubuntu 20.04.6 LTS   5.4.0-189-generic   containerd://1.6.33
```
YML manifest file: 
```
apiVersion: v1
kind: Pod
metadata:
  name: staticpod
  labels:
    app: nginx
spec:
  nodeName: worker2
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
```
```
kubectl create -f StaticPod.yml
```
```
kubectl get pods -o wide
```
Sample Output: 
```
NAME        READY   STATUS    RESTARTS   AGE   IP              NODE      NOMINATED NODE   READINESS GATES
staticpod   1/1     Running   0          31s   172.16.189.70   worker2   <none>           <none>
```
# Static Pod: Schedule pod(s) on specific node(s) using nodeSelector  
```
kubectl get nodes --show-labels
```
Sample Output: 
```
NAME      STATUS   ROLES           AGE   VERSION    LABELS
master1   Ready    control-plane   31d   v1.28.10   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=master1,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=,node.kubernetes.io/exclude-from-external-load-balancers=
worker1   Ready    <none>          31d   v1.28.10   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=worker1,kubernetes.io/os=linux
worker2   Ready    <none>          31d   v1.28.10   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=worker2,kubernetes.io/os=linux
worker3   Ready    <none>          66m   v1.28.11   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=worker3,kubernetes.io/os=linux
```
```
kubectl label nodes worker1 websrv=prod
kubectl label nodes worker3 websrv=prod
```
```
kubectl get nodes --show-labels
```
Sample Output: 
```
NAME      STATUS   ROLES           AGE   VERSION    LABELS
master1   Ready    control-plane   31d   v1.28.10   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=master1,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=,node.kubernetes.io/exclude-from-external-load-balancers=
worker1   Ready    <none>          31d   v1.28.10   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=worker1,kubernetes.io/os=linux,websrv=prod
worker2   Ready    <none>          31d   v1.28.10   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=worker2,kubernetes.io/os=linux
worker3   Ready    <none>          73m   v1.28.11   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=worker3,kubernetes.io/os=linux,websrv=prod
```
YML manifest file: 
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nodeselector-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      nodeSelector:
        websrv: prod
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
```
```
kubectl create -f NodeSelector-Deployment.yml
```
```
kubectl get pods -o wide
```
Sample Output: 
```
NAME                                       READY   STATUS    RESTARTS   AGE   IP               NODE      NOMINATED NODE   READINESS GATES
nodeselector-deployment-54cccc9c7f-5hr2b   1/1     Running   0          60s   172.16.235.131   worker1   <none>           <none>
nodeselector-deployment-54cccc9c7f-5sg7l   1/1     Running   0          60s   172.16.235.132   worker1   <none>           <none>
nodeselector-deployment-54cccc9c7f-j6cm4   1/1     Running   0          60s   172.16.182.0     worker3   <none>           <none>
```



