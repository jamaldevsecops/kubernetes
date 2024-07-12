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


