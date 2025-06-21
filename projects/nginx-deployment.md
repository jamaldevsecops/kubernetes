# Nginx Deployment with HTTPS in Kubernetes using MetalLB

## Pre-requisite:
- Kubernetes Cluster
- MetalLB
- Helm

---

### YAML Manifest for Nginx Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: default # You can change this to your specific namespace
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
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
        image: nginx:1.25 # Specify a stable image version
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1"
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
---
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-deployment
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 50
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: default
  annotations:		## change according to cloud load balancer
    metallb.universe.tf/address-pool: metallb-ip-pool
    metallb.universe.tf/loadBalancerIP: 192.168.20.135
    metallb.universe.tf/allow-shared-ip: "true"
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: LoadBalancer  	## change it to ClusterIP when using external load balancer, here I'm using MetalLB.
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800
```
Expose Nginx Service to http://k8sweb.apsissolutions.com

    Get the Service IP:
```bash
    root@master1:~/k8s_apps# kubectl get svc
    NAME            TYPE           CLUSTER-IP    EXTERNAL-IP      PORT(S)        AGE
    kubernetes      ClusterIP      10.96.0.1     <none>           443/TCP        98m
    nginx-service   LoadBalancer   10.110.1.47   192.168.20.131   80:32214/TCP   48m
```
    Ensure DNS Resolution: Point the domain k8sweb.apsissolutions.com to the external IP assigned by MetalLB for your Service (192.168.20.131).

Install Ingress Controller (NGINX Ingress Controller)

    Add the Ingress NGINX repository and install:
```bash
    helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
    helm repo update
    helm install nginx-ingress ingress-nginx/ingress-nginx
```
    Create the Ingress Resource (nginx-ingress.yml):
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: k8sweb.apsissolutions.com  # Domain name
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-service
            port:
              number: 80
```
    Apply the Ingress Resource:
```bash
    kubectl apply -f nginx-ingress.yml
```
    Test:

    Visit http://k8sweb.apsissolutions.com/.

Configure the Domain for HTTPS

    Create a Secret for TLS Certificate:
```bash
    kubectl create secret tls apsissolutions-com-ssl-certs \
      --cert=apsissolutions_CAChain.crt \
      --key=apsissolutions.com.key \
      --namespace=default
```
    Update the Ingress Resource for HTTPS:

    Open the nginx-ingress.yml file and add the TLS configuration:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  rules:
  - host: k8sweb.apsissolutions.com  # Domain name
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-service
            port:
              number: 80
  tls:
  - hosts:
    - k8sweb.apsissolutions.com
    secretName: apsissolutions-com-ssl-certs  # Name of the TLS secret
```
    Apply the Ingress Resource for HTTPS:
```bash
    kubectl apply -f nginx-ingress.yml
```
Conclusion

    Step 1: Deploy Nginx with a LoadBalancer Service.
    Step 2: Install and configure an Ingress controller.
    Step 3: Expose the Nginx service over HTTP and then configure HTTPS by creating a TLS secret.
    Step 4: Force HTTP to HTTPS redirection using NGINX annotations.
