To learn about Kubernetes Secrets for SSL termination in Ingress, let's break it down with an example that will cover the following:

1. **What is SSL Termination?**
2. **How to Use Kubernetes Secrets for SSL/TLS Certificates?**
3. **Configuring Ingress for SSL Termination.**

### 1. What is SSL Termination?

SSL (Secure Sockets Layer) termination is the process where the SSL-encrypted traffic is decrypted at the Ingress controller, and then the unencrypted traffic is sent to the application services. This setup offloads the decryption process from your application services, improving their performance and simplifying the management of SSL certificates.

### 2. Using Kubernetes Secrets for SSL/TLS Certificates

In Kubernetes, SSL/TLS certificates are stored as Secrets. A Kubernetes Secret is an object that contains sensitive data such as passwords, OAuth tokens, or SSH keys. When using SSL termination in Ingress, you'll store the SSL certificate and private key in a Kubernetes Secret of type `tls`.

#### Creating a Secret for SSL/TLS Certificates

Assume you have the following SSL certificate (`tls.crt`) and private key (`tls.key`):

- `tls.crt`: The certificate file.
- `tls.key`: The private key file.

You can create a Kubernetes Secret using the following command:

```bash
kubectl create secret tls my-ssl-cert \
  --cert=path/to/tls.crt \
  --key=path/to/tls.key
```

This will create a Secret named `my-ssl-cert` in the default namespace. The `--cert` option specifies the certificate file, and `--key` specifies the private key file.

Alternatively, you can define this Secret in a YAML file:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-ssl-cert
  namespace: default
type: kubernetes.io/tls
data:
  tls.crt: <base64-encoded-certificate>
  tls.key: <base64-encoded-private-key>
```

To get the base64-encoded values:

```bash
base64 -w 0 tls.crt
base64 -w 0 tls.key
```

Replace `<base64-encoded-certificate>` and `<base64-encoded-private-key>` with the actual base64 encoded values from the command output.

Apply the YAML file:

```bash
kubectl apply -f ssl-secret.yaml
```

### 3. Configuring Ingress for SSL Termination

Now that you have your SSL certificate stored in a Kubernetes Secret, you can configure an Ingress resource to use this Secret for SSL termination.

Here’s an example Ingress configuration that uses the `my-ssl-cert` Secret for SSL termination:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  tls:
  - hosts:
    - app1.apsis.com
    - app2.apsis.com
    secretName: my-ssl-cert
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

### Explanation:

- **tls**: The `tls` section specifies that the traffic for the `app1.apsis.com` and `app2.apsis.com` domains should use SSL termination with the certificate stored in the `my-ssl-cert` Secret.
- **secretName**: The name of the Secret that contains the SSL certificate and private key.
- **rules**: The `rules` section defines how the traffic should be routed based on the domain and path.

### How SSL Termination Works in Ingress

1. **Client Request**: A client makes an HTTPS request to `https://app1.apsis.com`.
2. **Ingress Controller**: The Ingress controller (e.g., NGINX Ingress) decrypts the SSL traffic using the certificate stored in the Secret (`my-ssl-cert`).
3. **Service Forwarding**: The decrypted traffic is then forwarded to the `app1-service` backend service on port 80.

### Summary

- **SSL Termination**: Offloads SSL decryption to the Ingress controller.
- **Kubernetes Secret**: Used to store SSL certificates and private keys.
- **Ingress Configuration**: Configured to use the Secret for SSL termination.

By using Kubernetes Secrets and Ingress, you can manage SSL certificates efficiently and ensure secure communication with your services.