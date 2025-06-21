### ConfigMap

**ConfigMaps** are used to store non-confidential data in key-value pairs. They can be used to configure application settings without changing the application code.

#### Creating a ConfigMap

1. **From a literal key-value pair**:
    ```sh
    kubectl create configmap my-config --from-literal=key1=value1 --from-literal=key2=value2
    ```

2. **From a file**:
    Create a file named `config.txt`:
    ```
    key1=value1
    key2=value2
    ```
    Then create the ConfigMap:
    ```sh
    kubectl create configmap my-config --from-file=config.txt
    ```

3. **From a YAML definition**:
    Create a file named `configmap.yaml`:
    ```yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: my-config
    data:
      key1: value1
      key2: value2
    ```
    Apply the file:
    ```sh
    kubectl apply -f configmap.yaml
    ```

#### Using a ConfigMap in a Pod

To use a ConfigMap in a Pod, you can reference it in the Pod's definition:

1. **As environment variables**:
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: my-pod
    spec:
      containers:
      - name: my-container
        image: busybox
        env:
        - name: KEY1
          valueFrom:
            configMapKeyRef:
              name: my-config
              key: key1
        - name: KEY2
          valueFrom:
            configMapKeyRef:
              name: my-config
              key: key2
        command: ["sh", "-c", "env"]
    ```

2. **As a volume**:
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: my-pod
    spec:
      containers:
      - name: my-container
        image: busybox
        volumeMounts:
        - name: config-volume
          mountPath: /etc/config
      volumes:
      - name: config-volume
        configMap:
          name: my-config
    ```

### Secret

**Secrets** are used to store sensitive information, such as passwords, OAuth tokens, and SSH keys. Secrets provide a way to securely pass sensitive data to Pods.

#### Creating a Secret

1. **From a literal key-value pair**:
    ```sh
    kubectl create secret generic my-secret --from-literal=username=myuser --from-literal=password=mypassword
    ```

2. **From a file**:
    Create a file named `secret.txt`:
    ```
    username=myuser
    password=mypassword
    ```
    Then create the Secret:
    ```sh
    kubectl create secret generic my-secret --from-file=secret.txt
    ```

3. **From a YAML definition**:
    Create a file named `secret.yaml`:
    ```yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: my-secret
    type: Opaque
    data:
      username: bXl1c2Vy   # Base64 encoded 'myuser'
      password: bXlwYXNzd29yZA==  # Base64 encoded 'mypassword'
    ```
    Apply the file:
    ```sh
    kubectl apply -f secret.yaml
    ```

#### Using a Secret in a Pod

To use a Secret in a Pod, you can reference it in the Pod's definition:

1. **As environment variables**:
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: my-pod
    spec:
      containers:
      - name: my-container
        image: busybox
        env:
        - name: USERNAME
          valueFrom:
            secretKeyRef:
              name: my-secret
              key: username
        - name: PASSWORD
          valueFrom:
            secretKeyRef:
              name: my-secret
              key: password
        command: ["sh", "-c", "env"]
    ```

2. **As a volume**:
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: my-pod
    spec:
      containers:
      - name: my-container
        image: busybox
        volumeMounts:
        - name: secret-volume
          mountPath: /etc/secret
      volumes:
      - name: secret-volume
        secret:
          secretName: my-secret
    ```

Let's create an example deployment that uses both ConfigMaps and Secrets.

### Step 1: Create a ConfigMap

Create a file named `configmap.yaml`:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-config
data:
  database_url: jdbc:mysql://mysql:3306/mydatabase
  log_level: INFO
```
Apply the ConfigMap:
```sh
kubectl apply -f configmap.yaml
```

### Step 2: Create a Secret

Create a file named `secret.yaml`:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque
data:
  username: bXl1c2Vy  # Base64 encoded 'myuser'
  password: bXlwYXNzd29yZA==  # Base64 encoded 'mypassword'
```
Apply the Secret:
```sh
kubectl apply -f secret.yaml
```

### Step 3: Create a Deployment

Create a file named `deployment.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-container
        image: busybox
        env:
        - name: DATABASE_URL
          valueFrom:
            configMapKeyRef:
              name: my-config
              key: database_url
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: my-config
              key: log_level
        - name: USERNAME
          valueFrom:
            secretKeyRef:
              name: my-secret
              key: username
        - name: PASSWORD
          valueFrom:
            secretKeyRef:
              name: my-secret
              key: password
        command: ["sh", "-c", "echo DB URL: $DATABASE_URL; echo Log Level: $LOG_LEVEL; echo Username: $USERNAME; echo Password: $PASSWORD; sleep 3600"]
```
Apply the Deployment:
```sh
kubectl apply -f deployment.yaml
```

### Verify the Deployment

1. **Check the status of the Deployment**:
    ```sh
    kubectl get deployments
    ```

2. **Check the Pods created by the Deployment**:
    ```sh
    kubectl get pods
    ```

3. **Inspect the logs of the Pod to verify the environment variables**:
    ```sh
    kubectl logs <pod-name>
    ```
    Replace `<pod-name>` with the name of your Pod. You should see output like:
    ```
    DB URL: jdbc:mysql://mysql:3306/mydatabase
    Log Level: INFO
    Username: myuser
    Password: mypassword
    ```


### Best Practices

1. **Avoid hardcoding sensitive information**: Use Secrets to manage sensitive data instead of hardcoding it in your code or config files.
2. **Restrict access**: Limit access to Secrets to only those Pods and users who need it.
3. **Monitor and audit**: Regularly monitor and audit the use of ConfigMaps and Secrets to ensure they are used securely and appropriately.
