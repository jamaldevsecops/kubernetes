Kubernetes common errors:

1. **CrashLoopBackOff**:
   - **Description**: A pod repeatedly crashes and restarts.
   - **Troubleshooting**:
     - Check pod logs: `kubectl logs <pod-name>`.
     - Describe the pod for more details: `kubectl describe pod <pod-name>`.
     - Investigate the application's start-up and initialization code.

2. **ImagePullBackOff**:
   - **Description**: Kubernetes cannot pull the container image from the registry.
   - **Troubleshooting**:
     - Verify the image name and tag.
     - Check the image registry credentials.
     - Ensure the image exists in the specified registry.

3. **Pending Pods**:
   - **Description**: Pods remain in the "Pending" state and are not scheduled.
   - Troubleshooting:
     - Check node resources (CPU, memory) to ensure there is enough capacity.
     - Ensure the nodes are labeled correctly if using node selectors or affinities.
     - Verify there are no taints on nodes that would prevent scheduling.

4. Node Not Ready:
   - Description: One or more nodes are in a "NotReady" state.
   - Troubleshooting:
     - Check node status: `kubectl describe node <node-name>`.
     - Review kubelet logs on the affected node.
     - Ensure the node has network connectivity.

5. Service Not Working
   - Description: Services are not accessible or routing traffic correctly.
   - Troubleshooting:
     - Check the service and endpoints: `kubectl get svc` and `kubectl get endpoints`.
     - Verify network policies and firewall rules.
     - Ensure the pods backing the service are healthy and running.

6. **Insufficient Resources**:
   - **Description**: Pods cannot be scheduled due to insufficient resources.
   - **Troubleshooting**:
     - Review resource requests and limits in pod specifications.
     - Scale the cluster by adding more nodes.

8. **PersistentVolume Claims Pending**:
   - **Description**: PVCs remain in a "Pending" state.
   - **Troubleshooting**:
     - Check if there are available PVs that match the PVC specifications.
     - Ensure the storage class exists and is configured correctly.
     - Verify that the underlying storage backend is healthy.

9. **Pod Stuck Terminating**:
   - **Description**: Pods get stuck in a "Terminating" state.
   - **Troubleshooting**:
     - Check for finalizers that might be preventing pod deletion.
     - Review the logs for shutdown hooks or long-running processes.
     - Force delete the pod if necessary: `kubectl delete pod <pod-name> --force --grace-period=0`.

10. **DNS Resolution Issues**:
    - **Description**: DNS lookups within the cluster fail.
    - **Troubleshooting**:
      - Check the DNS pod logs (e.g., CoreDNS): `kubectl logs <coredns-pod>`.
      - Ensure the DNS service is running: `kubectl get svc -n kube-system`.
      - Verify network policies and firewall rules do not block DNS traffic.
11. **Getting the following issue during Nginx Ingress Conroller installaiton.**
```bash
root@controlplane:~# helm install ingress-nginx ingress-nginx/ingress-nginx --version 4.12.0 --namespace ingress-nginx --create-namespace
Error: INSTALLATION FAILED: 1 error occurred:
	* Deployment.apps "ingress-nginx-controller" is invalid: spec.progressDeadlineSeconds: Invalid value: 0: must be greater than minReadySeconds
```
**Solutions:** Use a Known Working Version (e.g., 4.10.0)
```
helm search repo ingress-nginx/ingress-nginx --versions
helm install ingress-nginx ingress-nginx/ingress-nginx --version 4.10.0 --namespace ingress-nginx --create-namespace
```
Verify the Installation:   
```
kubectl get all -n ingress-nginx
```
Sample Outputs:  
```
NAME                                            READY   STATUS    RESTARTS   AGE
pod/ingress-nginx-controller-6c4dd4ddcc-w2ctx   1/1     Running   0          2m36s

NAME                                         TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)                      AGE
service/ingress-nginx-controller             LoadBalancer   10.102.211.58    192.168.20.131   80:32354/TCP,443:31577/TCP   2m36s
service/ingress-nginx-controller-admission   ClusterIP      10.105.146.108   <none>           443/TCP                      2m36s

NAME                                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/ingress-nginx-controller   1/1     1            1           2m36s

NAME                                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/ingress-nginx-controller-6c4dd4ddcc   1         1         1       2m36s

```
