# Install Helm

To install Helm, the Kubernetes package manager, follow the steps below based on your operating system. I’ll cover the installation for Linux, macOS, and Windows.

1. Install Helm on Linux (Ubuntu/Debian-based):

Using the Helm script:

You can install Helm via a simple script provided by the Helm team.
```
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```
This script will automatically download and install Helm for you.

Or, install manually via apt:

# Add the Helm GPG key and repository
```
curl https://baltocdn.com/helm/signing.asc | sudo apt-key add -
sudo apt-get install apt-transport-https --yes
echo "deb https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
```
# Update package index and install Helm
```
sudo apt-get update
sudo apt-get install helm
```
2. Install Helm on macOS:

Using Homebrew:

If you have Homebrew installed on your macOS, you can install Helm easily.
```
brew install helm
```
If you don’t have Homebrew, you can install it by running:
```
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```
Then run the brew install helm command as shown above.

3. Install Helm on Windows:

Using Chocolatey:

If you have Chocolatey installed, you can install Helm with:
```
choco install kubernetes-helm
```
Using Scoop:

Alternatively, if you have Scoop installed, use this command:
```
scoop install helm
```
4. Verify Helm Installation:

After installation, you can verify that Helm is installed by checking the version:
```
helm version
```
This should display the Helm version and confirm that it’s working properly.

5. Add Helm Repository (Optional):

To use Helm charts from the stable Helm repository, you may want to add it as a repository:
```
helm repo add stable https://charts.helm.sh/stable
helm repo update
```
6. Install a Helm Chart (Optional):

To install a chart using Helm, you can run a command like this (for example, installing the Nginx ingress controller):
```
helm install my-nginx ingress-nginx/ingress-nginx
```
That’s it! You now have Helm installed and ready to manage Kubernetes packages. Let me know if you need further assistance!


# Install Ingress Conteller
Kubernetes Ingress is a powerful resource that manages external access to services within a Kubernetes cluster, typically HTTP and HTTPS traffic. It allows you to expose multiple services under a single IP address and manage traffic routing based on hostnames, paths, and other rules.

### Key Concepts in Ingress

1. **Ingress Controller**: A Kubernetes resource that watches for Ingress objects and implements the rules defined in them. The NGINX Ingress Controller is one of the most commonly used controllers.

2. **Ingress Resource**: A Kubernetes resource that defines rules for routing external HTTP(S) traffic to services within the cluster.

3. **Path-Based Routing**: Route traffic to different services based on the URL path.

4. **Host-Based Routing**: Route traffic to different services based on the hostname.

5. **TLS/SSL Termination**: Secure the communication between clients and the services by terminating SSL/TLS at the Ingress controller.

6. **Annotations**: Provide additional configuration options, such as setting custom timeouts, enabling rewrites, or defining redirects.

### Basic Example: Exposing Two Services Using Ingress

Let's consider the scenario where you have two applications running on two different ports, 8080 and 8081. You want to access these applications using the following domain names:
- `https://app1.apsis.com`
- `https://app2.apsis.com`

We'll use the NGINX Ingress Controller to achieve this, with three replicas for each application.

#### Step 1: Install NGINX Ingress Controller
You can install the NGINX Ingress Controller using Helm:

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install my-nginx-ingress ingress-nginx/ingress-nginx
```

#### Step 2: Create Deployments and Services

```yaml
# Deployment for app1
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app1-deployment
  labels:
    app: app1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: app1
  template:
    metadata:
      labels:
        app: app1
    spec:
      containers:
      - name: app1-container
        image: my-app1-image
        command: ["/bin/sh", "-c"]
        args: ["echo 'Hello from App1' > /usr/share/nginx/html/index.html && exec nginx -g 'daemon off;'"]
        ports:
        - containerPort: 8080

---
# Service for app1
apiVersion: v1
kind: Service
metadata:
  name: app1-service
spec:
  selector:
    app: app1
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
```

```yaml
# Deployment for app2
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app2-deployment
  labels:
    app: app2
spec:
  replicas: 3
  selector:
    matchLabels:
      app: app2
  template:
    metadata:
      labels:
        app: app2
    spec:
      containers:
      - name: app2-container
        image: my-app2-image
        command: ["/bin/sh", "-c"]
        args: ["echo 'Hello from App2' > /usr/share/nginx/html/index.html && exec nginx -g 'daemon off;'"]
        ports:
        - containerPort: 8081

---
# Service for app2
apiVersion: v1
kind: Service
metadata:
  name: app2-service
spec:
  selector:
    app: app2
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8081
```

#### Step 3: Create an Ingress Resource

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: app1.apsis.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app1-service
            port:
              number: 80
  - host: app2.apsis.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app2-service
            port:
              number: 80
  tls:
  - hosts:
    - app1.apsis.com
    - app2.apsis.com
    secretName: tls-secret
```

### Explanation of the Manifest

- **Ingress Resource**: The `Ingress` object defines rules to route traffic based on the hostname.
- **TLS/SSL Termination**: The `tls` section is used to configure HTTPS for the specified hosts. The `tls-secret` should contain the SSL certificate and key.
- **Rewrite Target**: The `nginx.ingress.kubernetes.io/rewrite-target: /` annotation rewrites the URL path before forwarding it to the backend service.

### Step 4: Apply the YAML Files

```bash
kubectl apply -f app1-deployment.yaml
kubectl apply -f app2-deployment.yaml
kubectl apply -f app-ingress.yaml
```

### Step 5: Access Your Applications

After deploying the resources, you should be able to access your applications via:
- `https://app1.apsis.com`
- `https://app2.apsis.com`

### Advanced Ingress Features

1. **Rate Limiting**: Control the rate of requests allowed per client IP.
2. **Custom Error Pages**: Serve custom error pages.
3. **Basic Authentication**: Protect specific paths or services with basic authentication.
4. **Global Rate Limits**: Limit the number of requests per second for a given path across all clients.
5. **Advanced Load Balancing**: Use session affinity or weighted load balancing.
