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
        resources:
          limits:
            memory: "128Mi"
            cpu: "500m"
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
    nodePort: 30007  # Node Port. Optional field, by default and for convenience, the Kubernetes control plane will allocate a port from a range (default: 30000-32767)
```
```
kubectl create -f nginx-service.yml
```

