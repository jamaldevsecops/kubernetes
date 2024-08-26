Let's dive into some advanced Ingress features with practical examples:

### 1. **Rate Limiting**
   Rate limiting allows you to control the number of requests a client can make to your service. This can protect your application from being overwhelmed by too many requests.

**Example:**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/limit-connections: "20"
    nginx.ingress.kubernetes.io/limit-rpm: "30" # requests per minute
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
  tls:
  - hosts:
    - app1.apsis.com
    secretName: tls-secret
```

- **`nginx.ingress.kubernetes.io/limit-connections`**: Limits the number of connections per IP.
- **`nginx.ingress.kubernetes.io/limit-rpm`**: Limits the number of requests per minute per IP.

### 2. **Custom Error Pages**
   Serve custom error pages when your application returns an error (e.g., 404 Not Found).

**Example:**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/custom-http-errors: "404,500,502,503,504"
    nginx.ingress.kubernetes.io/default-backend: custom-error-service
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
  tls:
  - hosts:
    - app1.apsis.com
    secretName: tls-secret
```

- **`nginx.ingress.kubernetes.io/custom-http-errors`**: Specifies which HTTP errors will trigger the custom error page.
- **`nginx.ingress.kubernetes.io/default-backend`**: Defines a service that will serve custom error pages.

You need to deploy a separate service (`custom-error-service`) to serve these custom error pages.

### 3. **Basic Authentication**
   Protect your application by requiring a username and password.

**Example:**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/auth-type: "basic"
    nginx.ingress.kubernetes.io/auth-secret: "basic-auth-secret"
    nginx.ingress.kubernetes.io/auth-realm: "Authentication Required"
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
  tls:
  - hosts:
    - app1.apsis.com
    secretName: tls-secret
```

To create the basic auth secret:

```bash
htpasswd -c auth my-user
kubectl create secret generic basic-auth-secret --from-file=auth
```

### 4. **Advanced Load Balancing with Sticky Sessions**
   Use session affinity (sticky sessions) to ensure that requests from a client go to the same pod.

**Example:**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/affinity: "cookie"
    nginx.ingress.kubernetes.io/session-cookie-name: "route"
    nginx.ingress.kubernetes.io/session-cookie-hash: "sha1"
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
  tls:
  - hosts:
    - app1.apsis.com
    secretName: tls-secret
```

- **`nginx.ingress.kubernetes.io/affinity`**: Enables session affinity.
- **`nginx.ingress.kubernetes.io/session-cookie-name`**: Name of the cookie used for session affinity.
- **`nginx.ingress.kubernetes.io/session-cookie-hash`**: Hashing algorithm used for session affinity.

### 5. **Global Rate Limits**
   Apply rate limits globally across all clients and paths.

**Example:**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/limit-rps: "10" # requests per second
    nginx.ingress.kubernetes.io/limit-burst-multiplier: "3"
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
  tls:
  - hosts:
    - app1.apsis.com
    secretName: tls-secret
```

- **`nginx.ingress.kubernetes.io/limit-rps`**: Limits the number of requests per second globally.
- **`nginx.ingress.kubernetes.io/limit-burst-multiplier`**: Allows a burst of traffic before applying the rate limit.

### 6. **Redirect from HTTP to HTTPS**
   Automatically redirect HTTP traffic to HTTPS to ensure secure connections.

**Example:**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
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
  tls:
  - hosts:
    - app1.apsis.com
    secretName: tls-secret
```