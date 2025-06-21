# Kubernetes Service 
Source: https://kubernetes.io/docs/concepts/services-networking/service/

### Kubernetes Service (type: NodePort) 
The following is the contents of a deployment for nginx. 
```
cat nginx-deployment.yml
```
Sample Output: 
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
```
```
kubectl create -f nginx-deployment.yml
```
The following is the contents of a service for nginx.  
Source: https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport
```
cat nginx-service.yml
```
Sample Output: 
```
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80        # Service Port.
    targetPort: 80  # Pods Port. By default and for convenience, the `targetPort` is set to the same value as the `port` field.
    nodePort: 30190  # Node Port. Optional field, by default and for convenience, the Kubernetes control plane will allocate a port from a range (default: 30000-32767)
```
```
kubectl create -f nginx-service.yml
```
```
kubectl get pods -o wide
```
Sample Output: 
```
NAME                                READY   STATUS    RESTARTS   AGE   IP               NODE      NOMINATED NODE   READINESS GATES
nginx-deployment-57f79d6686-mb6zb   1/1     Running   0          13m   172.16.235.130   worker1   <none>           <none>
nginx-deployment-57f79d6686-vzs77   1/1     Running   0          13m   172.16.189.66    worker2   <none>           <none>
```
```
kubectl get service -o wide
```
Sample Output: 
```
NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE   SELECTOR
kubernetes      ClusterIP   10.96.0.1       <none>        443/TCP        72m   <none>
nginx-service   NodePort    10.98.134.231   <none>        80:30190/TCP   18m   app=nginx
```
```
kubectl get nodes -o wide
```
Sample Output: 
```
NAME      STATUS   ROLES           AGE   VERSION    INTERNAL-IP      EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
master1   Ready    control-plane   74m   v1.28.10   192.168.20.126   <none>        Ubuntu 20.04.2 LTS   5.4.0-137-generic   containerd://1.6.33
worker1   Ready    <none>          52m   v1.28.10   192.168.20.127   <none>        Ubuntu 20.04.6 LTS   5.4.0-182-generic   containerd://1.6.33
worker2   Ready    <none>          52m   v1.28.10   192.168.20.128   <none>        Ubuntu 20.04.6 LTS   5.4.0-182-generic   containerd://1.6.33
```
Access the service:  
http://192.168.20.126:30190/  
http://192.168.20.127:30190/  
http://192.168.20.128:30190/  











