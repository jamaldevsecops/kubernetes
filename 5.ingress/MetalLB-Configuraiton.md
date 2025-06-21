

# MetalLB Setup
![image](https://github.com/user-attachments/assets/a383137e-cfd9-42f6-8972-1c653548f7b4)

## Prerequisites
- Kubernetes Cluster: A running Kubernetes cluster with:
  - Master node: 192.168.20.126 (Installation script: https://tinyurl.com/k8s-master-node) 
  - Worker nodes: 192.168.20.127, 192.168.20.128 (Installation script: https://tinyurl.com/k8s-worker-node) 
- kubectl: Installed and configured on the machine you're using to interact with the cluster (e.g., on the master node or your local machine).
- Root or sudo access: Required for installing tools like Helm and modifying system files (e.g., /etc/hosts).
- Network Access: Ensure all nodes can communicate, and the IP range 192.168.20.131–192.168.20.135 is available and not used by other services.
- Internet Access: Required to download manifests, Helm charts, and container images.

All commands below should be run on the machine where kubectl is configured, typically the master node (192.168.20.126), unless specified otherwise.

## ***1. Install Helm***  
Helm is a package manager for Kubernetes that simplifies the deployment of applications like the Nginx Ingress Controller.

Steps
Download the Helm installation script:
1. Download the Helm installation script:
```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
```
2. Set execute permissions:
```bash
chmod 700 get_helm.sh
```
3. Run the installation script:
```bash
./get_helm.sh
```
4. Verify Helm installation:
```bash
helm version
```
- Expected output: Something like version.BuildInfo {Version:"v3.x.x", ...}.
- This confirms Helm is installed correctly.

Notes
- The script installs the latest stable version of Helm (v3.x.x as of May 2025).
- If you encounter issues, ensure curl is installed (sudo apt-get install curl on Debian/Ubuntu or sudo yum install curl on CentOS/RHEL).
- Helm requires kubectl to be configured to communicate with your cluster.

***2. Install MetalLB***  
MetalLB provides LoadBalancer services in bare-metal Kubernetes clusters, which is necessary since your cluster doesn't have a cloud provider with native LoadBalancer support. We'll configure MetalLB to use the IP pool 192.168.20.131–192.168.20.135.

Steps
1. Enable Static ARP:
```bash
kubectl get ns
kubectl get configmap -n kube-system
kubectl edit configmap kube-proxy -n kube-system 
```
```plain
<output omitted> 
    ipvs:
      excludeCIDRs: null
      minSyncPeriod: 0s
      scheduler: ""
      strictARP: true
      syncPeriod: 0s
      tcpFinTimeout: 0s
      tcpTimeout: 0s
      udpTimeout: 0s
<output omitted> 
```

2. Apply the MetalLB manifests:
```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.9/config/manifests/metallb-native.yaml
```
- This deploys MetalLB's controller, speaker pods, and other required resources.

3. Defines a pool of IPs that MetalLB is allowed to assign to services of type LoadBalancer and tells MetalLB to advertise the IPs from that pool using Layer 2 (ARP):
```bash
cat > metallb-config.yaml << EOF
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.20.131-192.168.20.135
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default
  namespace: metallb-system
spec:
  ipAddressPools:
  - default-pool
EOF
```
```bash
kubectl apply -f metallb-config.yaml
kubectl get ipaddresspools -n metallb-system
kubectl get l2advertisements -n metallb-system
```
- This enables MetalLB to broadcast (ARP) the availability of those IPs on the local network.
- Any service of type LoadBalancer in your cluster can now be assigned an external IP from the default-pool.


4. Verify MetalLB installation:
```bash
kubectl get pods -n metallb-system -o wide
```
- Expected output: You should see pods like controller-xxx and speaker-xxx in the Running state, e.g.:

```plain
NAME                         READY   STATUS    RESTARTS   AGE   IP               NODE           NOMINATED NODE   READINESS GATES
controller-7dcb87658-snbr6   1/1     Running   0          18m   172.16.189.65    worker2        <none>           <none>
speaker-gd4xb                1/1     Running   0          18m   192.168.20.127   worker1        <none>           <none>
speaker-rjqjw                1/1     Running   0          18m   192.168.20.126   controlplane   <none>           <none>
speaker-sx7rm                1/1     Running   0          18m   192.168.20.128   worker2        <none>           <none>
```
- The speaker pods run on each node (one per node, so expect two speaker pods for your two worker nodes).

Notes
- MetalLB version v0.14.9 is the latest stable release as of my knowledge update. If a newer version is available, replace v0.14.9 with the latest version from the MetalLB GitHub repository.
- Ensure the IP range 192.168.20.131–192.168.20.135 is not used by other devices or services in your network.
- If pods fail to start, check logs with kubectl logs -n metallb-system <pod-name> to diagnose issues (e.g., network misconfiguration).

***3. Install Nginx Ingress Controller***  
The Nginx Ingress Controller handles external HTTP/HTTPS traffic and routes it to services within the cluster. We'll install it using Helm and configure it to use MetalLB's LoadBalancer with a specific IP (192.168.20.131).

Steps
1. Add the ingress-nginx Helm repository:
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
```

2. Update Helm repositories:
```bash
helm repo update
```
- This ensures you have the latest chart versions.

3. Install the Nginx Ingress Controller:
```bash
helm install ingress-nginx ingress-nginx/ingress-nginx --namespace ingress-nginx --create-namespace
```
or, statically assign IP

```bash
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=LoadBalancer \
  --set controller.service.loadBalancerIP=192.168.20.131
```
* --namespace ingress-nginx: Installs the controller in the ingress-nginx namespace, creating it if it doesn't exist.
* --set controller.service.type=LoadBalancer: Configures the service as a LoadBalancer, which MetalLB will handle.
* --set controller.service.loadBalancerIP=192.168.20.131: Assigns the specific IP from the MetalLB pool.

4. Verify the Nginx Ingress Controller:
```bash
kubectl get pods -n ingress-nginx
```
- Expected output: A pod like ingress-nginx-controller-xxx in the Running state, e.g.:
```plain
NAME                                        READY   STATUS    RESTARTS   AGE
ingress-nginx-controller-7f49467bd9-zsxr2   1/1     Running   0          2m17s
```
```bash
kubectl get svc -n ingress-nginx
```
- Expected output: A service with an external IP of 192.168.20.131, e.g.:
```plain
NAME                                 TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.107.21.51    192.168.20.131   80:32441/TCP,443:30820/TCP   2m33s
ingress-nginx-controller-admission   ClusterIP      10.108.175.61   <none>           443/TCP                      2m33s
```

Notes
- The Nginx Ingress Controller pod may take a few minutes to become Running as it pulls the container image.
- If the EXTERNAL-IP is not assigned, ensure MetalLB is running correctly (kubectl get pods -n metallb-system) and the ConfigMap is applied.
- The loadBalancerIP (192.168.20.131) must be within the MetalLB pool defined earlier.

***4. Deploy Nginx Webserver with HPA, Service, and Ingress***  
We'll deploy an Nginx webserver with a Deployment, Horizontal Pod Autoscaler (HPA) to scale based on CPU usage, a ClusterIP Service, and an Ingress resource to route external traffic via the Nginx Ingress Controller.

Steps
1. Create a TLS secret for apsissolutions.com
```bash
kubectl create secret tls apsissolutions.com --cert=CA_chain.crt --key=apsissolutions.com.key -n default
```
2. Create a YAML file for the Nginx webserver resources:
```bash
cat <<EOF > nginx-webserver.yaml
# Nginx Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: default
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 20%
      maxSurge: 30%
  minReadySeconds: 10
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
        image: nginx:1.20
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "256Mi" # 25% of 1 GiB memory. 
            cpu: "250m" # 25% of 1 core.
          limits:
            memory: "512Mi"
            cpu: "500m"
---
# Horizontal Pod Autoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-deployment
  minReplicas: 5
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70

---
# Service
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: default
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80

---
# Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - mynginx.apsissolutions.com
    secretName: apsissolutions.com 
  rules:
  - host: mynginx.apsissolutions.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-service
            port:
              number: 80
EOF
```
- Deployment: Runs 2 replicas of the Nginx webserver with resource requests and limits.
- HPA: Scales the Deployment between 2 and 5 replicas based on 70% CPU utilization.
- Service: Exposes the Nginx pods internally via a ClusterIP on port 80.
- Ingress: Routes external traffic from mynginx.apsissolutions.com to the Service, using the Nginx Ingress Controller.

3. Apply the resources:
```bash
kubectl apply -f nginx-webserver.yaml
```

4. Verify the deployment:
```bash
kubectl get pods,svc,ingress,hpa -n default
```
- Expected output:
```plain
NAME                                    READY   STATUS    RESTARTS   AGE
pod/nginx-deployment-848656f9fd-5svsf   1/1     Running   0          23m
pod/nginx-deployment-848656f9fd-9kd2c   1/1     Running   0          23m
pod/nginx-deployment-848656f9fd-bzsfc   1/1     Running   0          25m
pod/nginx-deployment-848656f9fd-cx876   1/1     Running   0          23m
pod/nginx-deployment-848656f9fd-vkgb2   1/1     Running   0          25m

NAME                    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/kubernetes      ClusterIP   10.96.0.1       <none>        443/TCP   3h30m
service/nginx-service   ClusterIP   10.110.192.20   <none>        80/TCP    25m

NAME                                      CLASS   HOSTS                        ADDRESS          PORTS     AGE
ingress.networking.k8s.io/nginx-ingress   nginx   mynginx.apsissolutions.com   192.168.20.131   80, 443   25m

NAME                                            REFERENCE                     TARGETS              MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/nginx-hpa   Deployment/nginx-deployment   cpu: <unknown>/70%   5         20        5          
```

Notes
- The nginx:latest image is used for simplicity. For production, pin a specific version (e.g., nginx:1.27).
- The HPA requires a metrics server to be running in the cluster to monitor CPU usage. If it's not installed, you may need to deploy it:
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```
- The Ingress uses mynginx.apsissolutions.com as a placeholder domain. You'll configure DNS or /etc/hosts for testing.

***5. Testing the Setup***  
To verify that everything works, follow these steps to access the Nginx webserver and test the setup.

Steps
1. Get the LoadBalancer IP:
```bash
kubectl get svc -n ingress-nginx
```
- Look for the EXTERNAL-IP of the ingress-nginx-controller service, which should be 192.168.20.131.
- Example output:
```plain
NAME                                 TYPE           CLUSTER-IP      EXTERNAL-IP       PORTS                      AGE
ingress-nginx-controller             LoadBalancer   10.96.123.456   192.168.20.131    80:31000/TCP,443:32000/TCP   10m
```

2. Configure DNS or hosts file:
- Since mynginx.apsissolutions.com is a test domain, add an entry to your local /etc/hosts file (or equivalent) on the machine you're testing from.
- Run (with sudo/root privileges):
```bash
echo "192.168.20.131 webserver.apsissolutions.com" | sudo tee -a /etc/hosts
```
- This maps mynginx.apsissolutions.com to the LoadBalancer IP.

3. Test access to the Nginx webserver:
```bash
curl https://webserver.apsissolutions.com
```
- Expected output: The default Nginx welcome page HTML, starting with:
```plain
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
```
- Alternatively, open http://mynginx.apsissolutions.com in a web browser, and you should see the Nginx welcome page.

4. Test Horizontal Pod Autoscaler (optional):
- To simulate load and trigger the HPA, use a load-testing tool like hey or ab.
- Install hey (if not installed):
```bash
go install github.com/rakyll/hey@latest
```
- Note: Requires Go installed (sudo apt-get install golang or equivalent).
- Generate load:
```bash
~/go/bin/hey -n 10000 -c 100 http://mynginx.apsissolutions.com
```
- This sends 10,000 requests with 100 concurrent clients.
- Monitor the HPA:
```bash
kubectl get hpa -n default -w
```
- Watch for the REPLICAS column to increase (up to 5) if CPU usage exceeds 70%.
- Example output during scaling:
```plain
NAME                        REFERENCE                    TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
nginx-webserver-hpa         Deployment/nginx-webserver   85%/70%   2         5         3          15m
```

5. Verify pod scaling:
```bash
kubectl get pods -n default
```
- If the HPA triggered, you should see additional nginx-webserver pods (up to 5).

Notes
- If curl fails, ensure:
  * The Ingress Controller pod is running (kubectl get pods -n ingress-nginx).
  * The /etc/hosts entry is correct.
  * The firewall allows traffic to port 80 on 192.168.20.131.
- If the HPA doesn't scale, verify the metrics server is running:
```bash
kubectl get pods -n kube-system | grep metrics-server
```
  * If not present, install it as noted above.
- For production, configure a proper DNS record instead of /etc/hosts.

Troubleshooting
- MetalLB issues:
  * Pods not running: Check logs (kubectl logs -n metallb-system <pod-name>).
  * No external IP assigned: Ensure the ConfigMap is correct and IPs are not in use elsewhere.
- Nginx Ingress Controller issues:
  * Pod not starting: Check logs (kubectl logs -n ingress-nginx <pod-name>).
  * No external IP: Verify MetalLB is functioning and the loadBalancerIP is in the pool.
- Nginx webserver issues:
  * Pods not running: Check kubectl describe pod -n default <pod-name> for errors (e.g., image pull issues).
  * Ingress not routing: Ensure the ingressClassName: nginx matches the Ingress Controller and the host is correctly configured.
- HPA not scaling:
  * Verify metrics server is running.
  * Check HPA events: kubectl describe hpa -n default nginx-webserver-hpa.

Cleanup (Optional)
If you need to remove the resources for testing or redeployment:
```bash 
# Remove Nginx webserver resources
kubectl delete -f nginx-webserver.yaml

# Uninstall Nginx Ingress Controller
helm uninstall ingress-nginx -n ingress-nginx
kubectl delete namespace ingress-nginx

# Uninstall MetalLB
kubectl delete -f https://raw.githubusercontent.com/metallb/metallb/v0.14.9/config/manifests/metallb-native.yaml
kubectl delete -f https://raw.githubusercontent.com/metallb/metallb/v0.14.9/config/manifests/namespace.yaml
kubectl delete configmap config -n metallb-system

# Remove Helm (if desired)
rm -f get_helm.sh
sudo rm -rf /usr/local/bin/helm
```

Additional Notes
- Security: For production, secure the Ingress with TLS by adding a certificate (e.g., via cert-manager).
- Persistence: The Nginx webserver uses the default Nginx image without persistent storage. For production, consider adding a PersistentVolumeClaim.
- Monitoring: Install monitoring tools like Prometheus and Grafana to track cluster performance and HPA behavior.
- Backup: Save the nginx-webserver.yaml file and these instructions in a version control system or secure location for future reference.
- Updates: Periodically check for updates to MetalLB, Helm, and the Nginx Ingress Controller to stay on supported versions.
