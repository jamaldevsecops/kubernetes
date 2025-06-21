To automate the management of TLS certificates in your Kubernetes cluster, you can use **cert-manager**, a popular Kubernetes add-on. Cert-manager can automatically issue, renew, and manage TLS certificates from various Certificate Authorities (CAs), including Let's Encrypt.

### Choosing the Right Version of Cert-Manager

1. **Compatibility with Kubernetes Version**:
   - Ensure that the version of cert-manager you choose is compatible with your Kubernetes cluster version. Cert-manager releases are aligned with Kubernetes releases, so you should check the cert-manager documentation for compatibility details.

2. **Issuer Type**:
   - **Let's Encrypt**: If you're using Let's Encrypt for issuing certificates, you'll configure cert-manager to use an `Issuer` or `ClusterIssuer` for Let's Encrypt.
   - **Self-Signed Certificates**: If you're using self-signed certificates for internal use, cert-manager can handle these as well.
   - **Commercial CAs**: Cert-manager can also work with commercial CAs if you have a paid certificate service.

3. **Deployment Mode**:
   - **Helm Chart**: The easiest way to deploy cert-manager is through a Helm chart, which handles installation and upgrades seamlessly.
   - **Manifest Files**: Alternatively, you can apply the cert-manager manifests directly if you're not using Helm.

### Installing Cert-Manager

Here’s how to install cert-manager using Helm:

1. **Add the Jetstack Helm Repository**:

   ```bash
   helm repo add jetstack https://charts.jetstack.io
   helm repo update
   ```

2. **Install Cert-Manager**:

   ```bash
   helm install cert-manager jetstack/cert-manager \
     --namespace cert-manager \
     --create-namespace \
     --version v1.14.0 \
     --set installCRDs=true
   ```

   - `--version v1.14.0`: Replace with the latest compatible version for your Kubernetes setup.
   - `--set installCRDs=true`: Ensures that the necessary Custom Resource Definitions (CRDs) are installed.

3. **Verify the Installation**:

   ```bash
   kubectl get pods --namespace cert-manager
   ```

   You should see the cert-manager pods running.

### Basic Setup of Let's Encrypt Issuer

If you're using Let's Encrypt, set up a `ClusterIssuer` to issue certificates:

1. **Create a ClusterIssuer for Let's Encrypt**:

   ```yaml
   apiVersion: cert-manager.io/v1
   kind: ClusterIssuer
   metadata:
     name: letsencrypt-prod
   spec:
     acme:
       server: https://acme-v02.api.letsencrypt.org/directory
       email: your-email@example.com
       privateKeySecretRef:
         name: letsencrypt-prod
       solvers:
       - http01:
           ingress:
             class: nginx
   ```

   - `email`: Replace with your email for notifications about certificate renewal issues.
   - `ingress.class`: Should match the class used by your Ingress controller (e.g., `nginx`).

2. **Apply the `ClusterIssuer`**:

   ```bash
   kubectl apply -f cluster-issuer.yaml
   ```

### Using Cert-Manager with Ingress

To use cert-manager with an Ingress resource:

1. **Example Ingress with Cert-Manager Annotations**:

   ```yaml
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: app-ingress
     annotations:
       cert-manager.io/cluster-issuer: "letsencrypt-prod"
   spec:
     tls:
     - hosts:
       - app1.apsis.com
       secretName: app1-tls
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

2. **Apply the Ingress**:

   ```bash
   kubectl apply -f ingress.yaml
   ```

Cert-manager will automatically request a certificate from Let's Encrypt, store it in the `app1-tls` secret, and renew it before it expires.

### Key Features of Cert-Manager

- **Automated Certificate Renewal**: Cert-manager automatically renews certificates before they expire.
- **Multiple Issuers**: You can define multiple issuers (e.g., Let's Encrypt, self-signed) and choose the appropriate one for each use case.
- **ACME Challenges**: Cert-manager supports HTTP-01, DNS-01, and TLS-ALPN-01 challenges for ACME-based issuers like Let's Encrypt.
- **Webhook Support**: Cert-manager provides webhooks for additional validation and custom logic.

Here are examples for using cert-manager to handle both **self-signed certificates** and certificates issued by a **commercial CA**.

### 1. Self-Signed Certificates

#### Step 1: Create a Self-Signed Issuer

A `SelfSignedIssuer` generates certificates that are signed by the issuer itself.

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: selfsigned-issuer
  namespace: default
spec:
  selfSigned: {}
```

Apply the manifest:

```bash
kubectl apply -f selfsigned-issuer.yaml
```

#### Step 2: Create a Certificate Resource

Next, create a `Certificate` resource that uses the `SelfSignedIssuer` to generate a self-signed certificate.

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: selfsigned-cert
  namespace: default
spec:
  secretName: selfsigned-cert-tls
  duration: 24h # Certificate valid for 24 hours
  renewBefore: 1h # Renew 1 hour before expiry
  commonName: app1.apsis.com
  dnsNames:
  - app1.apsis.com
  issuerRef:
    name: selfsigned-issuer
    kind: Issuer
```

Apply the manifest:

```bash
kubectl apply -f selfsigned-certificate.yaml
```

This will create a self-signed certificate stored in a secret called `selfsigned-cert-tls`.

#### Step 3: Use the Certificate in an Ingress Resource

Finally, reference the generated certificate in an Ingress resource:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app1-ingress
  namespace: default
  annotations:
    cert-manager.io/issuer: "selfsigned-issuer"
spec:
  tls:
  - hosts:
    - app1.apsis.com
    secretName: selfsigned-cert-tls
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

Apply the Ingress:

```bash
kubectl apply -f app1-ingress.yaml
```

### 2. Commercial CA Certificates

#### Step 1: Obtain Certificates from a Commercial CA

First, you need to obtain the following from your Commercial CA:

- **Certificate**: The signed certificate for your domain.
- **Private Key**: The private key associated with the certificate.
- **CA Bundle**: The certificate chain or intermediate certificates provided by your CA.

#### Step 2: Create a Secret with the CA Certificates

Create a secret in Kubernetes to store the CA-issued certificate, private key, and CA bundle.

```bash
kubectl create secret tls commercial-ca-cert-tls \
  --cert=path/to/your-cert.pem \
  --key=path/to/your-key.pem \
  --namespace default
```

Alternatively, use a YAML manifest:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: commercial-ca-cert-tls
  namespace: default
type: kubernetes.io/tls
data:
  tls.crt: <base64-encoded-certificate>
  tls.key: <base64-encoded-private-key>
```

Encode the certificate and key files to base64:

```bash
base64 -w 0 your-cert.pem
base64 -w 0 your-key.pem
```

Replace `<base64-encoded-certificate>` and `<base64-encoded-private-key>` with the respective base64 strings.

Apply the manifest:

```bash
kubectl apply -f commercial-ca-cert-secret.yaml
```

#### Step 3: Create an Ingress Resource

Finally, use this secret in your Ingress resource:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app2-ingress
  namespace: default
spec:
  tls:
  - hosts:
    - app2.apsis.com
    secretName: commercial-ca-cert-tls
  rules:
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

Apply the Ingress:

```bash
kubectl apply -f app2-ingress.yaml
```

### Summary

- **Self-Signed Certificates**: Useful for internal services where you don't need a trusted CA. Managed with a `SelfSignedIssuer`.
- **Commercial CA Certificates**: Required for public-facing services where a trusted CA issues the certificate. Managed by storing the CA-provided certificate and key in a Kubernetes secret.

By following these steps, you can manage both self-signed and commercial CA certificates within your Kubernetes cluster using cert-manager.