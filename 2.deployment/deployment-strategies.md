Kubernetes offers several deployment strategies to manage the release of new versions of applications. These strategies help ensure smooth transitions with minimal downtime and can accommodate various application needs. Here are the main Kubernetes deployment strategies:

### 1. **Recreate Deployment**
In this strategy, all old versions of the application are stopped before new ones are created.

**Pros:**
- Simple to implement.
- No overlap between old and new versions.

**Cons:**
- Causes downtime.

**Example:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: recreate-deployment
spec:
  strategy:
    type: Recreate
  replicas: 3
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp:2.0
```

### 2. **Rolling Update**
The rolling update strategy gradually replaces old versions of pods with new ones. This ensures that some instances of the application remain available during the update process.

**Pros:**
- Minimal downtime.
- Can control the speed of the update.

**Cons:**
- Requires careful monitoring.

**Example:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rolling-update-deployment
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  replicas: 3
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp:2.0
```

### 3. **Blue-Green Deployment**
In blue-green deployments, two environments (blue and green) are maintained. The new version (green) is deployed alongside the old version (blue). Once the new version is verified, traffic is switched to the new version.

**Pros:**
- No downtime.
- Easy rollback.

**Cons:**
- Requires more resources.

**Example:**
- Deploy the new version in a separate environment.
- Switch the service to point to the new version.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp-green
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
```

### 4. **Canary Deployment**
In canary deployments, a small subset of users is directed to the new version of the application. If the new version performs well, the rollout continues until all users are on the new version.

**Pros:**
- Controlled, gradual rollout.
- Can test new features in production.

**Cons:**
- More complex to manage.

**Example:**
- Deploy a small number of pods with the new version.
- Gradually increase the number of new version pods.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: canary-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
      version: canary
  template:
    metadata:
      labels:
        app: myapp
        version: canary
    spec:
      containers:
      - name: myapp
        image: myapp:2.0
```

### Real-life Example of Rolling Update with nginx

Let's use the nginx image for a rolling update deployment. Assume we start with nginx:1.19 and want to update to nginx:1.20.

**Initial Deployment:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
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
        image: nginx:1.19
        ports:
        - containerPort: 80
```

**Update to nginx:1.20:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
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
        image: nginx:1.20
        ports:
        - containerPort: 80
```

In this example, the rolling update will replace one pod at a time, ensuring that there are always at least two pods running during the update. The `maxUnavailable` and `maxSurge` parameters control the update process, where `maxUnavailable` specifies the maximum number of pods that can be unavailable during the update, and `maxSurge` specifies the maximum number of pods that can be created above the desired number of replicas.


### Rolling Update with Percentage and minReadySeconds

To control a rolling update using percentages and to ensure that new pods are ready before terminating old ones, you can use the `maxUnavailable`, `maxSurge`, and `minReadySeconds` settings in your deployment strategy.

- **`maxUnavailable`**: This specifies the maximum number of pods that can be unavailable during the update process, defined as either an integer or a percentage.
- **`maxSurge`**: This specifies the maximum number of pods that can be created above the desired number of replicas, defined as either an integer or a percentage.
- **`minReadySeconds`**: This specifies the minimum number of seconds a newly created pod should be ready without any of its containers crashing to be considered available.

Here's how you can define a rolling update with these parameters:

### Example: Rolling Update with Percentage and minReadySeconds

Let's assume we have a deployment of `nginx` with 10 replicas. We want to update from `nginx:1.19` to `nginx:1.20`, allowing up to 20% of the pods to be unavailable during the update and allowing up to 30% surge pods. We also want each new pod to be ready for at least 10 seconds before considering it available.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 20%       # Allow up to 20% of pods to be unavailable during the update
      maxSurge: 30%             # Allow up to 30% more pods than the desired number of replicas
  minReadySeconds: 10           # Each new pod must be ready for at least 10 seconds before being considered available
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
```

### Explanation:

- **`maxUnavailable: 20%`**: With 10 replicas, up to 2 pods (20% of 10) can be unavailable during the update process.
- **`maxSurge: 30%`**: With 10 replicas, up to 3 additional pods (30% of 10) can be created during the update process, allowing a total of 13 pods running temporarily.
- **`minReadySeconds: 10`**: Each new pod must be ready for at least 10 seconds before Kubernetes considers it available, which helps ensure stability before proceeding with the next update step.

### How it Works:
1. Kubernetes creates new pods with the updated version (`nginx:1.20`).
2. It ensures each new pod runs for at least 10 seconds without crashing (`minReadySeconds: 10`).
3. It gradually terminates old pods (`nginx:1.19`) while respecting the `maxUnavailable` and `maxSurge` limits.
4. The process continues until all old pods are replaced by new pods.

This configuration ensures a controlled, gradual rollout with a balance between availability and the speed of deployment.


Rollback in Kubernetes refers to reverting a deployment to a previous version if the current deployment has issues. This can be crucial for maintaining application stability. Kubernetes stores the history of deployments, allowing you to quickly revert to a previous version if needed.

### Rolling Back a Deployment

To demonstrate a rollback, let's use the `nginx` deployment example where we update from `nginx:1.19` to `nginx:1.20`. If the new version has issues, we can roll back to the previous version.

#### Step-by-Step Example:

1. **Initial Deployment:**
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: nginx-deployment
   spec:
     replicas: 10
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
           image: nginx:1.19
           ports:
           - containerPort: 80
   ```

2. **Updating to a New Version (nginx:1.20):**
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: nginx-deployment
   spec:
     replicas: 10
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
   ```

3. **Rollback to the Previous Version (nginx:1.19):**

   To rollback to the previous version, use the following command:

   ```sh
   kubectl rollout undo deployment/nginx-deployment
   ```

   This command reverts the deployment to the previous version. If you need to rollback to a specific revision, you can check the revision history and specify the revision number.

4. **Check Revision History:**
   ```sh
   kubectl rollout history deployment/nginx-deployment
   ```

   This command lists the revision history of the deployment. Each revision has a unique number, which you can use for a specific rollback.

5. **Rollback to a Specific Revision:**
   ```sh
   kubectl rollout undo deployment/nginx-deployment --to-revision=1
   ```

   Replace `1` with the desired revision number.

### Example Commands and Outputs

- **Update the Deployment:**
  ```sh
  kubectl apply -f nginx-deployment.yaml
  ```

- **Check Rollout Status:**
  ```sh
  kubectl rollout status deployment/nginx-deployment
  ```

- **Check Revision History:**
  ```sh
  kubectl rollout history deployment/nginx-deployment
  ```

  **Sample Output:**
  ```
  deployment.apps/nginx-deployment
  REVISION  CHANGE-CAUSE
  1         <none>
  2         <none>
  ```

- **Rollback to Previous Revision:**
  ```sh
  kubectl rollout undo deployment/nginx-deployment
  ```

- **Rollback to a Specific Revision:**
  ```sh
  kubectl rollout undo deployment/nginx-deployment --to-revision=1
  ```

### Ensuring a Safe Rollback

- **Monitor the Rollback:** Use `kubectl rollout status` to monitor the status of the rollback and ensure that it completes successfully.
- **Check Logs:** Check the logs of the pods to diagnose issues that caused the rollback.
- **Update Strategy:** Consider refining your update strategy (e.g., canary deployment) to reduce the risk of issues in future updates.

Rolling back a deployment is a powerful feature that ensures you can quickly restore service if a new deployment has issues. By monitoring deployments and maintaining good version control practices, you can minimize the impact of any issues that arise.


