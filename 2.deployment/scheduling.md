# Manual Scheduling 

### Manual Schedule Using nodeName  
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
### Manual Schedule Using using nodeSelector  
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
kubectl label nodes worker1 environment=production
kubectl label nodes worker3 environment=production
```
```
kubectl get nodes --show-labels
```
Sample Output: 
```
NAME      STATUS   ROLES           AGE   VERSION    LABELS
master1   Ready    control-plane   31d   v1.28.10   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=master1,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=,node.kubernetes.io/exclude-from-external-load-balancers=
worker1   Ready    <none>          31d   v1.28.10   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=worker1,kubernetes.io/os=linux,environment=production
worker2   Ready    <none>          31d   v1.28.10   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=worker2,kubernetes.io/os=linux
worker3   Ready    <none>          73m   v1.28.11   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=worker3,kubernetes.io/os=linux,environment=production
```
```
kubectl get nodes -l environment=production
```
Sample Output:
```
NAME      STATUS   ROLES    AGE   VERSION
worker1   Ready    <none>   31d   v1.28.10
worker3   Ready    <none>   87m   v1.28.11
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
        environment: production
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
### Manual Scheduling with nodeAffinity (RequiredDuringSchedulingIgnoredDuringExecution)
Ref: https://kubernetes.io/docs/tasks/configure-pod-container/assign-pods-nodes-using-node-affinity/  
```RequiredDuringSchedulingIgnoredDuringExecution:``` This means that the rule is mandatory for scheduling a pod, but once the pod is scheduled, the rule is not enforced. For example, if a node label changes after the pod is already running, the pod will not be rescheduled.  

### operator:  
The operator field is used in labelSelector to define the relationship between the label key and values. Common operators include:  

```In:``` The label key must have one of the specified values.  
```NotIn:``` The label key must not have any of the specified values.  
```Exists:``` The label key must exist.  
```DoesNotExist:``` The label key must not exist.  

YML manifest file: 
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-node-affinity-deployment 
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
      affinity: 
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: environment 
                    operator: In 
                    values: 
                      - production

      containers:
      - name: nginx
        image: nginx 
        ports:
        - containerPort: 80
```
```
kubectl create -f nodeaffinity.yml 
kubectl get deployment
```
Sample Output: 
```
NAME                             READY   UP-TO-DATE   AVAILABLE   AGE
nginx-node-affinity-deployment   2/2     2            2           57s
```
```
kubectl get pods -o wide
```
Sample Output: 
```
NAME                                              READY   STATUS    RESTARTS   AGE   IP               NODE      NOMINATED NODE   READINESS GATES
nginx-node-affinity-deployment-55b7bb75ff-fqpf8   1/1     Running   0          75s   172.16.235.133   worker1   <none>           <none>
nginx-node-affinity-deployment-55b7bb75ff-kmpsk   1/1     Running   0          75s   172.16.182.1     worker3   <none>           <none>
```

YML manifest file: 
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-node-affinity-example2-deployment
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
      affinity: 
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: datacenter 
                    operator: In 
                    values: 
                      - us-east-1 
      containers:
      - name: nginx
        image: nginx 
        ports:
        - containerPort: 80
```
```
kubectl create -f nodeaffinity.yml 
kubectl get deployments
```
Sample Output: 
```
NAME                                      READY   UP-TO-DATE   AVAILABLE   AGE
nginx-node-affinity-example2-deployment   0/2     2            0           39s
```
```
kubectl get pods -o wide
```
Sample Output: 
```
NAME                                                      READY   STATUS    RESTARTS   AGE   IP               NODE      NOMINATED NODE   READINESS GATES
nginx-node-affinity-example2-deployment-89ff99b54-p4nj4   0/1     Pending   0          85s   <none>           <none>    <none>           <none>
nginx-node-affinity-example2-deployment-89ff99b54-rgjkx   0/1     Pending   0          85s
```   
```
kubectl label nodes worker2 datacenter=us-east-1
kubectl get pods -o wide
```
Sample Output: 
```
NAME                                                      READY   STATUS    RESTARTS   AGE     IP               NODE      NOMINATED NODE   READINESS GATES
nginx-node-affinity-example2-deployment-89ff99b54-p4nj4   1/1     Running   0          4m16s   172.16.189.72    worker2   <none>           <none>
nginx-node-affinity-example2-deployment-89ff99b54-rgjkx   1/1     Running   0          4m16s   172.16.189.71    worker2   <none>           <none>
```

### Manual Scheduling with nodeAffinity (PreferredDuringSchedulingIgnoredDuringExecution)
```PreferredDuringSchedulingIgnoredDuringExecution:``` This means that the rule is preferred but not mandatory for scheduling a pod. The scheduler will try to place the pod on a node that satisfies the rule, but if no such node is available, the pod will still be scheduled. Again, once the pod is scheduled, the rule is not enforced.  

YML manifest file: 
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-node-affinity-example3-deployment 
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
      affinity: 
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - preference:
                matchExpressions:
                  - key: disktype
                    operator: In
                    values: 
                      - ssd
              weight: 1
          
      containers:
      - name: nginx
        image: nginx 
        ports:
        - containerPort: 80
```
```
kubectl create -f nodeaffinity.yml
kubectl get deployment
```
Sample Output: 
```
NAME                                      READY   UP-TO-DATE   AVAILABLE   AGE
nginx-node-affinity-example3-deployment   2/2     2            2           66s
```
```
kubectl get pods -o wide
```
Sample Output: 
```
NAME                                                       READY   STATUS    RESTARTS   AGE   IP               NODE      NOMINATED NODE   READINESS GATES
nginx-node-affinity-example3-deployment-746dccfd85-fvn4n   1/1     Running   0          98s   172.16.235.134   worker1   <none>           <none>
nginx-node-affinity-example3-deployment-746dccfd85-kqmjg   1/1     Running   0          98s   172.16.189.73    worker2   <none>           <none>
```
```
kubectl label nodes worker2 environment=development
```
```
kubectl get nodes --show-labels
```
Sample Output: 
```
NAME      STATUS   ROLES           AGE   VERSION    LABELS
master1   Ready    control-plane   32d   v1.28.10   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=master1,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=,node.kubernetes.io/exclude-from-external-load-balancers=
worker1   Ready    <none>          32d   v1.28.10   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,environment=production,kubernetes.io/arch=amd64,kubernetes.io/hostname=worker1,kubernetes.io/os=linux
worker2   Ready    <none>          32d   v1.28.10   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,environment=development,kubernetes.io/arch=amd64,kubernetes.io/hostname=worker2,kubernetes.io/os=linux
worker3   Ready    <none>          19h   v1.28.11   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,environment=production,kubernetes.io/arch=amd64,kubernetes.io/hostname=worker3,kubernetes.io/os=linux
```
YML manifest file: 
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-node-affinity-example4-deployment 
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
      affinity: 
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - preference:
                matchExpressions:
                  - key: disktype
                    operator: In
                    values: 
                      - ssd
              weight: 1
            
            - preference:
                matchExpressions:
                  - key: environment 
                    operator: In 
                    values:
                      - development 
              weight: 2
          
      containers:
      - name: nginx
        image: nginx 
        ports:
        - containerPort: 80
```
```
kubectl create -f nodeaffinity.yml
kubectl get pods -o wide
```
Sample Output: 
```
NAME                                                      READY   STATUS    RESTARTS   AGE   IP              NODE      NOMINATED NODE   READINESS GATES
nginx-node-affinity-example4-deployment-9f7d784dd-5266j   1/1     Running   0          41s   172.16.189.74   worker2   <none>           <none>
nginx-node-affinity-example4-deployment-9f7d784dd-hfsmj   1/1     Running   0          41s   172.16.189.75   worker2   <none>           <none>
```

YAML manifest file:  
```NotIn:``` The label key must not have any of the specified values.
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-node-affinity-example5-deployment 
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
      affinity: 
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - preference:
                matchExpressions:
                  - key: disktype
                    operator: NotIn
                    values: 
                      - ssd
              weight: 1
          
      containers:
      - name: nginx
        image: nginx 
        ports:
        - containerPort: 80
```
```
kubectl create -f nodeaffinity.yml 
kubectl get deployments
```
Sample Output:
```
NAME                                      READY   UP-TO-DATE   AVAILABLE   AGE
nginx-node-affinity-example5-deployment   2/2     2            2           10s
```
```
kubectl get pods -o wide
```
Sample Output:
```
NAME                                                       READY   STATUS    RESTARTS   AGE   IP               NODE      NOMINATED NODE   READINESS GATES
nginx-node-affinity-example5-deployment-865b979dfb-9jwg7   1/1     Running   0          28s   172.16.189.76    worker2   <none>           <none>
nginx-node-affinity-example5-deployment-865b979dfb-g5qkm   1/1     Running   0          28s   172.16.235.135   worker1   <none>           <none>
```



# Taints and Tolerations

Taints and tolerations in Kubernetes are used in various real-life scenarios to ensure that pods are scheduled appropriately on nodes. Here are some practical examples:

•	Operators:  
```Exists```: Matches taints with the specified key, regardless of value.  
```Equal```: Matches taints with the specified key and value.  

•	Effects:  
```NoSchedule```: Prevents scheduling pods that do not tolerate the taint.  
```PreferNoSchedule```: Tries to avoid scheduling pods that do not tolerate the taint but is not strict.  
```NoExecute```: Evicts existing pods that do not tolerate the taint and prevents new pods from being scheduled on the node.  

### 1. Isolating Critical Workloads

Suppose you have a set of nodes dedicated to running critical workloads and you want to prevent non-critical workloads from being scheduled on these nodes.  

Taint the Node
```
kubectl taint nodes worker1 critical=true:NoSchedule
kubectl taint nodes worker2 critical=true:NoSchedule
```
Tolerate the Taint in Deployment
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: critical-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: critical-app
  template:
    metadata:
      labels:
        app: critical-app
    spec:
      tolerations:
      - key: "critical"
        operator: "Equal"
        value: "true"
        effect: "NoSchedule"
      containers:
      - name: critical-app
        image: my-critical-app:latest
```
### 2. Maintenance Nodes  

You may want to drain nodes for maintenance without disrupting running workloads immediately but prevent new pods from being scheduled.  

Taint the Nodes  
```
kubectl taint nodes worker3 maintenance=true:NoExecute
```
Tolerate the Taint in Deployment (if some pods can tolerate being on maintenance nodes)  
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tolerating-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: tolerating-app
  template:
    metadata:
      labels:
        app: tolerating-app
    spec:
      tolerations:
      - key: "maintenance"
        operator: "Equal"
        value: "true"
        effect: "NoExecute"
      containers:
      - name: tolerating-app
        image: my-tolerating-app:latest
```
### 3. Dedicated Nodes for GPU Workloads

Nodes with GPUs are expensive and should be dedicated to workloads that require them.  

Taint the Nodes  
```
kubectl taint nodes worker1 gpu=true:NoSchedule
kubectl taint nodes worker2 gpu=true:NoSchedule
```
Tolerate the Taint in Deployment  
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gpu-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: gpu-app
  template:
    metadata:
      labels:
        app: gpu-app
    spec:
      tolerations:
      - key: "gpu"
        operator: "Equal"
        value: "true"
        effect: "NoSchedule"
      containers:
      - name: gpu-app
        image: my-gpu-app:latest
        resources:
          limits:
            nvidia.com/gpu: 1
```
### 4. Prefer Avoiding Nodes with Issues

Nodes in certain zones might have intermittent network issues. You might prefer to avoid scheduling pods there but allow it if necessary.  

Taint the Nodes  
```
kubectl taint nodes worker1 network-issues=true:PreferNoSchedule
kubectl taint nodes worker2 network-issues=true:PreferNoSchedule
```
Tolerate the Taint in Deployment  
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: resilient-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: resilient-app
  template:
    metadata:
      labels:
        app: resilient-app
    spec:
      tolerations:
      - key: "network-issues"
        operator: "Exists"
        effect: "PreferNoSchedule"
      containers:
      - name: resilient-app
        image: my-resilient-app:latest
```




