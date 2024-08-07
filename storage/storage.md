# Kubernetes Storage
```
cat pv-pod.yml
```
```
apiVersion: v1
kind: Pod
metadata:
  name: webapp
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
    env:
    - name: LOG_HANDLERS
      value: file
    volumeMounts:
    - mountPath: /var/log/nginx
      name: pv-vol

  volumes:
  - name: pv-vol
    persistentVolumeClaim:
      claimName: nginx-pv-claim
```
```
cat pv-deployment.yml
```
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
        env:
        - name: LOG_HANDLERS
          value: file
        volumeMounts:
        - mountPath: /var/log/nginx
          name: pv-vol
      volumes:
      - name: pv-vol
        persistentVolumeClaim:
          claimName: nginx-pv-claim
```


```
cat pv-retain-policy.yml
```
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nginx-pv-volume
  labels:
    type: local
spec:
  persistentVolumeReclaimPolicy: Retain
  accessModes:
    - ReadWriteMany
  capacity:
    storage: 1Gi
  hostPath:
    path: "/mnt/data"
```
```
cat pv-delete-policy.yml
```
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nginx-pv-volume
  labels:
    type: local
spec:
  persistentVolumeReclaimPolicy: Delete
  accessModes:
    - ReadWriteMany
  capacity:
    storage: 1Gi
  hostPath:
    path: "/mnt/data"
```
```
cat pv-recycle-policy.yml
```
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nginx-pv-volume
  labels:
    type: local
spec:
  persistentVolumeReclaimPolicy: Delete
  accessModes:
    - ReadWriteMany
  capacity:
    storage: 1Gi
  hostPath:
    path: "/mnt/data"
```


```
cat pvc.yml
```
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx-pv-claim
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
```



```
cat pv-service.yml
```
```
apiVersion: v1
kind: Service
metadata:
  name: webapp-service
spec:
  type: NodePort
  selector:
    app: webapp
  ports:
  - protocol: TCP
    port: 80        
    targetPort: 80  
    nodePort: 30190
```

