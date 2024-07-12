# Static Pod: Schedule a pod on a specific node using nodeName

```
kubectl get node -o wide
```
Sample Output: 
```
NAME      STATUS   ROLES           AGE   VERSION    INTERNAL-IP      EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
master1   Ready    control-plane   31d   v1.28.10   192.168.20.126   <none>        Ubuntu 20.04.2 LTS   5.4.0-189-generic   containerd://1.6.33
worker1   Ready    <none>          31d   v1.28.10   192.168.20.127   <none>        Ubuntu 20.04.6 LTS   5.4.0-189-generic   containerd://1.6.33
worker2   Ready    <none>          31d   v1.28.10   192.168.20.128   <none>        Ubuntu 20.04.6 LTS   5.4.0-189-generic   containerd://1.6.33
```
YML manifest file: 
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
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
      nodeName: worker2
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
```
```
kubectl create -f StaticPod-Deployment.yml
```
```
kubectl get pods -o wide
```
Sample Output: 
```
NAME                                READY   STATUS    RESTARTS   AGE    IP              NODE      NOMINATED NODE   READINESS GATES
nginx-deployment-76f479f548-ckxtn   1/1     Running   0          4m5s   172.16.189.68   worker2   <none>           <none>
nginx-deployment-76f479f548-pvzm4   1/1     Running   0          4m5s   172.16.189.69   worker2   <none>           <none>
nginx-deployment-76f479f548-r5b4x   1/1     Running   0          4m5s   172.16.189.67   worker2   <none>           <none>
```


