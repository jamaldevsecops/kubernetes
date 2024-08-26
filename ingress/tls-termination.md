### TLS Termination with Advanced Features in Kubernetes

TLS termination is a process where encrypted traffic is decrypted at the load balancer or Ingress controller before being forwarded to the backend services. This is useful for offloading the CPU-intensive decryption process from your application servers.

Here's how you can implement TLS termination in Kubernetes using the NGINX Ingress controller, along with some advanced features like automatic HTTP to HTTPS redirection, custom SSL certificates, and client certificate authentication (mTLS).

### Prerequisites

- **NGINX Ingress Controller** installed on your Kubernetes cluster.
- **TLS Certificate** and **Private Key** stored as a Kubernetes secret.

### Step 1: Create a TLS Secret

You need to create a Kubernetes secret that contains your TLS certificate and private key.

```bash
kubectl create secret tls tls-secret --cert=path/to/tls.crt --key=path/to/tls.key
```

### Step 2: Define an Ingress Resource with TLS Termination

This basic example shows how to set up TLS termination with automatic redirection from HTTP to HTTPS.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
  - hosts:
    - app1.apsis.com
    - app2.apsis.com
    secretName: tls-secret
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
```

### Advanced Features

#### 1. **Custom SSL Certificates per Host**
   You might want to use different SSL certificates for different hosts. You can define multiple TLS blocks in the Ingress resource.

**Example:**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-tls-ingress
  annotations:
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  tls:
  - hosts:
    - app1.apsis.com
    secretName: tls-secret-app1
  - hosts:
    - app2.apsis.com
    secretName: tls-secret-app2
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
```

#### 2. **Client Certificate Authentication (mTLS)**
   Mutual TLS (mTLS) ensures that both the client and server authenticate each other. This adds an extra layer of security, ensuring that only trusted clients can connect.

**Steps:**

1. **Create a Secret for the CA Certificate**:

    ```bash
    kubectl create secret generic ca-secret --from-file=ca.crt=path/to/ca.crt
    ```

2. **Configure Ingress for mTLS**:

    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: mTLS-ingress
      annotations:
        nginx.ingress.kubernetes.io/auth-tls-secret: "default/ca-secret"
        nginx.ingress.kubernetes.io/auth-tls-verify-client: "on"
        nginx.ingress.kubernetes.io/auth-tls-verify-depth: "2"
        nginx.ingress.kubernetes.io/auth-tls-pass-certificate-to-upstream: "true"
    spec:
      tls:
      - hosts:
        - secure-app.apsis.com
        secretName: tls-secret
      rules:
      - host: secure-app.apsis.com
        http:
          paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: secure-app-service
                port:
                  number: 80
    ```

- **`nginx.ingress.kubernetes.io/auth-tls-secret`**: Specifies the CA certificate secret used to verify client certificates.
- **`nginx.ingress.kubernetes.io/auth-tls-verify-client`**: Enables client certificate verification.
- **`nginx.ingress.kubernetes.io/auth-tls-pass-certificate-to-upstream`**: Passes the verified client certificate to the backend service.

#### 3. **HTTP to HTTPS Redirection with Custom Error Pages**
   Combine HTTP to HTTPS redirection with custom error pages in case of SSL errors or client authentication failures.

**Example:**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: advanced-ingress
  annotations:
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/custom-http-errors: "495,496,497" # SSL errors
    nginx.ingress.kubernetes.io/default-backend: custom-error-service
spec:
  tls:
  - hosts:
    - app1.apsis.com
    secretName: tls-secret
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
```

### Step 3: Apply the YAML Files

```bash
kubectl apply -f tls-ingress.yaml
kubectl apply -f multi-tls-ingress.yaml
kubectl apply -f mtls-ingress.yaml
kubectl apply -f advanced-ingress.yaml
```

### Key Takeaways

- **TLS Termination**: Offloads the SSL/TLS decryption to the Ingress controller, freeing up resources on your application servers.
- **Custom SSL Certificates**: Different certificates can be used for different domains.
- **Client Certificate Authentication (mTLS)**: Provides a stronger level of security by ensuring that both client and server authenticate each other.
- **HTTP to HTTPS Redirection**: Ensures that all traffic to your services is encrypted.
- **Custom Error Pages**: Allows you to serve user-friendly error pages in case of SSL-related issues.

These advanced features give you fine-grained control over how traffic is handled in your Kubernetes cluster, enhancing security, performance, and user experience.