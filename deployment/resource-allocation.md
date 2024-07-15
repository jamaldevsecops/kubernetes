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

### Resource Requests and Limits
Resource Requests: This is the amount of CPU and memory that Kubernetes guarantees to a container. The scheduler uses this information to decide on which node to place the pod.

Resource Limits: This is the maximum amount of CPU and memory that a container is allowed to use. If a container tries to exceed its limit, it may be throttled (CPU) or killed (memory).

Setting Resource Limits
Resource requests and limits are defined in the pod or container specification within your YAML configuration file.

Example
Here's an example of setting resource requests and limits for a container in a pod:
```
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
```
kubectl apply -f resource-demo-pod.yaml
```
