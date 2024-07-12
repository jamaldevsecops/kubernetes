# Kubernetes Deployment

1. List Nodes with Detailed Information:
```
kubectl get nodes -o wide
```

2. Create a Deployment named nginx-deployment with the nginx Image and 2 Replicas:
```
kubectl create deployment nginx-depoyment --image=nginx --replicas=2
```

3. Scale up the nginx-deployment Deployment to 5 Replicas:
```
kubectl scale deployment nginx-depoyment --replicas=5
```

4. Scale down the nginx-deployment Deployment to 3 Replicas:
```
kubectl scale deployment nginx-depoyment --replicas=3
```

5. List Deployments:
```
kubectl get deployments
```

6. Describe the nginx-deployment Deployment:
```
kubectl describe deployment nginx-depoyment
```

7. List Pods:
```
kubectl get pods
```

8. Describe the Pod with Name nginx-deployment-7c5dc5d946-hgszx:
```
kubectl describe pods nginx-depoyment-7c5dc5d946-hgszx
```

9. Create YAML menifest file from imperative command
```
kubectl create deployment redis-depoyment --image=redis --replicas=2 --dry-run=client -o yaml > redis-deployment.yaml
```

10. Create an YAML menifest file and create a deployment.   

```
nano redis-deployment 
```
add the following contents
```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: redis-deployment
  name: redis-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis-deployment
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 25%
      maxSurge: 25%
  template:
    metadata:
      labels:
        app: redis-deployment
    spec:
      containers:
      - image: redis:7.2.5
        name: redis
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
status: {}

```
```
kubectl apply -f redis-deployment
```

11. Retrieve and Save the YAML Manifest of the running nginx-deployment Deployment:
```
kubectl get deployment nginx-depoyment -o yaml > nginx-deployment.yaml
```

