# Understanding Memory and CPU Units in Kubernetes

### Memory

Memory in Kubernetes is typically expressed in bytes. The most common units are:

	•	Mi (Mebibytes): 1 MiB = 2^20 bytes = 1,048,576 bytes
	•	Gi (Gibibytes): 1 GiB = 2^30 bytes = 1,073,741,824 bytes
	•	Ki (Kibibytes): 1 KiB = 2^10 bytes = 1,024 bytes

### CPU

CPU resources are measured in cores. The units are:

	•	m (millicores): 1 core = 1000 millicores
	•	cores: Fractional values like 0.5 or 1.5 cores

Example Calculations

	•	Memory:
	•	64Mi means 64 Mebibytes.
	•	In bytes: 64 MiB = 64 * 2^20 bytes = 64 * 1,048,576 bytes = 67,108,864 bytes.
	•	CPU:
	•	250m means 250 millicores.
	•	In cores: 250 millicores = 250 / 1000 cores = 0.25 cores.

# Resource Requests and Limits

**Resource Requests:** This is the amount of CPU and memory that Kubernetes guarantees to a container. The scheduler uses this information to decide on which node to place the pod.

**Resource Limits:** This is the maximum amount of CPU and memory that a container is allowed to use. If a container tries to exceed its limit, it may be throttled (CPU) or killed (memory).

Example
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-demo
  namespace: default
spec:
  containers:
  - name: resource-demo-container
    image: nginx
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```
```sh
kubectl apply -f resource-demo-pod.yaml
kubectl top pod resource-demo
```
**Best Practices**
1. Set realistic resource requests and limits: Base these on observed usage and benchmarking to ensure efficient use of resources.
2. Monitor and adjust: Continuously monitor resource usage and adjust requests and limits as necessary.
3. Use resource quotas: In larger clusters, use ResourceQuotas to limit the total resource consumption of a namespace to prevent resource exhaustion.

# Resource Quotas
To ensure fair resource distribution among different teams or applications, you can use ResourceQuotas.
Example
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: resource-quota
  namespace: default
spec:
  hard:
    pods: "10"
    requests.cpu: "4"
    requests.memory: "8Gi"
    limits.cpu: "8"
    limits.memory: "16Gi"
```
```sh
kubectl apply -f resource-quota.yaml
kubectl get resourcequota -n default
```
This ensures that the default namespace cannot exceed the specified resource limits.


### Real Life Scenario 1: Web Application
For a typical web application, you might have a deployment with an NGINX frontend and a Node.js backend.

**NGINX Frontend**
NGINX is usually lightweight and doesn’t need a lot of resources.

```yaml
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
      containers:
      - name: nginx
        image: nginx:latest
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
```
**requests:** Guarantees that each NGINX pod gets at least 64Mi of memory and 100m of CPU.
**limits:** Ensures that each NGINX pod does not exceed 128Mi of memory and 200m of CPU.

**Node.js Backend**
Node.js might require more resources, especially under load.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nodejs-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nodejs
  template:
    metadata:
      labels:
        app: nodejs
    spec:
      containers:
      - name: nodejs
        image: node:14
        resources:
          requests:
            memory: "256Mi"
            cpu: "200m"
          limits:
            memory: "512Mi"
            cpu: "500m"
```
**requests:** Guarantees that each Node.js pod gets at least 256Mi of memory and 200m of CPU.
**limits:** Ensures that each Node.js pod does not exceed 512Mi of memory and 500m of CPU.

### Real Life Scenario 2: Machine Learning Application
A machine learning application might use TensorFlow and could require significant resources.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ml-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ml-app
  template:
    metadata:
      labels:
        app: ml-app
    spec:
      containers:
      - name: tensorflow
        image: tensorflow/tensorflow:latest
        resources:
          requests:
            memory: "2Gi"
            cpu: "1"
          limits:
            memory: "4Gi"
            cpu: "2"
```

### Real Life Scenario 3: Database Application
For a database application like MySQL, you need to ensure that it has enough memory and CPU to handle queries efficiently.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:5.7
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "password"
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "1"
```

### Real Life Scenario 4: CI/CD Pipeline
For a CI/CD pipeline using Jenkins, which can be resource-intensive during builds:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jenkins
  template:
    metadata:
      labels:
        app: jenkins
    spec:
      containers:
      - name: jenkins
        image: jenkins/jenkins:lts
        resources:
          requests:
            memory: "2Gi"
            cpu: "1"
          limits:
            memory: "4Gi"
            cpu: "2"
```
