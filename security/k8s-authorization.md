Authorization in Kubernetes (K8S) is a key part of its security model, determining whether a user or service account has the necessary permissions to perform an action on a Kubernetes resource. Here’s an overview of how authorization works in Kubernetes:

### 1. **RBAC (Role-Based Access Control)**
   - **Roles and RoleBindings**: 
     - **Roles** define permissions within a specific namespace. They contain rules that specify the actions allowed on resources (e.g., `get`, `list`, `create` on `pods`, `services`, etc.).
     - **RoleBindings** associate a Role with a user, group, or service account, granting the permissions defined in the Role to that entity within a specific namespace.
   Example:
   ```yaml
   # Role definition in a namespace
   kind: Role
   apiVersion: rbac.authorization.k8s.io/v1
   metadata:
     namespace: default
     name: pod-reader
   rules:
   - apiGroups: [""]
     resources: ["pods"]
     verbs: ["get", "list"]
   ```
	
```yaml
   # RoleBinding to bind the Role to a user
   kind: RoleBinding
   apiVersion: rbac.authorization.k8s.io/v1
   metadata:
     name: read-pods
     namespace: default
   subjects:
   - kind: User
     name: devtechlead
     apiGroup: rbac.authorization.k8s.io
   roleRef:
     kind: Role
     name: pod-reader
     apiGroup: rbac.authorization.k8s.io
```
   - **ClusterRoles and ClusterRoleBindings**:
     - **ClusterRoles** are similar to Roles but apply across the entire cluster, not just within a single namespace.
     - **ClusterRoleBindings** associate a ClusterRole with users, groups, or service accounts, granting permissions cluster-wide.
	Example:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pod-viewer
rules:
- apiGroups: [""]
  #
  # "" indicates the core API group. You can specify other API groups, like "apps" or "extensions", if needed.
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: pod-viewer-binding
subjects:
- kind: User
  name: devtechlead
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: pod-viewer
  apiGroup: rbac.authorization.k8s.io
```


### 2. **ABAC (Attribute-Based Access Control)**
   - In ABAC, policies are defined in a file where each policy specifies attributes like user, resource, verb, etc.
   - ABAC is less commonly used compared to RBAC due to its complexity and lack of native integration into Kubernetes.
   - Example of an ABAC policy:
     ```json
     {
       "apiVersion": "abac.authorization.kubernetes.io/v1beta1",
       "kind": "Policy",
       "spec": {
         "user": "alice",
         "namespace": "default",
         "resource": "pods",
         "verb": "get"
       }
     }
     ```

### 3. **Webhook Authorization**
   - In Webhook authorization, requests are sent to a remote service to determine if an action is allowed.
   - The external service is responsible for returning an allow/deny decision based on its own logic and policies.

### 4. **Node Authorization**
   - Node authorization restricts access to resources based on the node where the request originates. It’s mainly used for kubelet communication.

### 5. **AlwaysAllow and AlwaysDeny**
   - These modes are mostly for testing and are not recommended for production environments. `AlwaysAllow` grants all requests, while `AlwaysDeny` denies all requests.

### **Authorization Flow in Kubernetes**
   1. **Authentication**: First, the user or service account making the request is authenticated.
   2. **Authorization**: After authentication, Kubernetes checks if the authenticated entity is authorized to perform the requested action on the target resource using one or more of the above methods.
   3. **Admission Control**: If authorized, the request then goes through admission controllers for additional checks before being allowed.
---
### **Real-Life Example Scenario for Roles and RoleBindings**

Let’s consider a real-life scenario where you need to use `Roles` and `RoleBindings` to manage access within a specific namespace in your Kubernetes cluster.

### **Scenario: Multi-Tenant Application Environment**

Imagine you have a Kubernetes cluster hosting a multi-tenant application where each tenant has its own namespace. Each namespace contains resources such as deployments, services, and secrets specific to that tenant. The tenants have their own teams of developers who need access to manage the resources within their respective namespaces but should not be able to view or modify resources in other tenants' namespaces.

#### **Objective:**
You want to allow the developers of each tenant to manage their resources (like deployments, services, and secrets) within their specific namespace while ensuring they do not have access to other namespaces.

### **Solution Using Roles and RoleBindings:**

#### 1. **Define a `Role`:**
   - Create a `Role` in each tenant’s namespace that grants permissions to manage specific resources like deployments, services, and secrets.

#### 2. **Create a `RoleBinding`:**
   - Bind the `Role` to the specific developers or service accounts that belong to that tenant, giving them the necessary permissions within their namespace.

### **Example: Tenant A’s Namespace**

Let's say Tenant A has a namespace called `crm-webapp1`. The following example shows how to create a `Role` and `RoleBinding` for the developers of Tenant A.

### **Role YAML**
This `Role` grants permissions to manage deployments, services, and secrets within the `crm-webapp1` namespace.
```yaml
cat crm-team-roles.yaml
```
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: crm-webapp1
  name: crm-developer-role
  labels:
    role: developer
    app: crm-webapp1
  annotations:
    createdBy: "admin"
    description: "Role for CRM web app developers to manage deployments, services, and secrets."
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "create", "update", "delete"]
- apiGroups: [""]
  resources: ["services"]
  verbs: ["get", "list", "create", "update", "delete"]
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list", "create", "update", "delete"]
```
```yaml
kubectl apply -f crm-team-roles.yaml 
kubectl get role -n crm-webapp1
```
Example Output:
```yaml
NAME                 CREATED AT
crm-developer-role   2024-08-20T11:45:15Z
```
```yaml
kubectl describe role crm-developer-role -n crm-webapp1
```
Example Ouput: 
```yaml
Name:         crm-developer-role
Labels:       app=crm-webapp1
              role=developer
Annotations:  createdBy: admin
              description: Role for CRM web app developers to manage deployments, services, and secrets.
PolicyRule:
  Resources         Non-Resource URLs  Resource Names  Verbs
  ---------         -----------------  --------------  -----
  secrets           []                 []              [get list create update delete]
  services          []                 []              [get list create update delete]
  deployments.apps  []                 []              [get list create update delete]
```




### **RoleBinding YAML**
This `RoleBinding` binds the `tenant-a-developer` `Role` to a group called `tenant-a-devs`, allowing members of this group to manage resources within the `tenant-a` namespace.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: tenant-a-developer-binding
  namespace: tenant-a
subjects:
- kind: Group
  name: tenant-a-devs
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: tenant-a-developer
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f tenant-a-role.yaml
kubectl apply -f tenant-a-rolebinding.yaml
```

### **Outcome:**

- **Namespace Isolation**: Tenant A’s developers have the necessary permissions to manage resources within their own namespace (`tenant-a`), but they cannot access or modify resources in other tenants’ namespaces.
- **Granular Access Control**: Each tenant’s team can be assigned their own roles and bindings, ensuring that permissions are tailored to their specific needs, following the principle of least privilege.
- **Security**: This approach helps in maintaining strict security boundaries between different tenants in a multi-tenant Kubernetes cluster.
This setup ensures that different teams or tenants can work independently within their own namespaces, reducing the risk of accidental interference or unauthorized access to resources belonging to other teams.
---
### **Real-Life Example Scenario for ClusterRole and ClusterRoleBinding**
Let’s explore a real-life scenario where you might use `ClusterRole` and `ClusterRoleBinding` to manage access within a Kubernetes cluster.

### **Scenario: Centralized Logging System**

Imagine you have a Kubernetes cluster that hosts multiple applications across different namespaces. Each application is managed by different teams, and the cluster has a centralized logging system deployed using tools like Elasticsearch, Fluentd, and Kibana (often referred to as the EFK stack). The logging system collects logs from all applications across the cluster, which can be critical for monitoring, debugging, and compliance.

#### **Objective:**
You want to provide the logging team with read-only access to all logs across the cluster so they can monitor application logs but ensure they cannot modify or delete any resources.

### **Solution Using ClusterRole and ClusterRoleBinding:**

#### 1. **Define the `ClusterRole`:**
   - Create a `ClusterRole` that allows the logging team to view logs (`get`, `list`, `watch` pods and their logs) across all namespaces.

#### 2. **Create a `ClusterRoleBinding`:**
   - Bind this `ClusterRole` to the logging team’s user accounts or service accounts, granting them the necessary permissions.

### **ClusterRole YAML**
This `ClusterRole` grants permissions to view logs by allowing access to `pods` and `pods/log` resources.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: log-viewer
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
```

### **ClusterRoleBinding YAML**
This `ClusterRoleBinding` binds the `log-viewer` `ClusterRole` to the `logging-team` group, giving all users in that group access to view logs across the cluster.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: logging-team-binding
subjects:
- kind: Group
  name: logging-team
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: log-viewer
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f log-viewer-clusterrole.yaml
kubectl apply -f logging-team-binding.yaml
```

### **Outcome:**

- **Security**: The logging team can access all logs for monitoring and analysis, but they cannot interfere with the operation of the pods or modify any resources.
- **Granular Access Control**: Other teams, such as developers or operations, can have their own roles and bindings tailored to their specific needs, ensuring that permissions are tightly controlled and aligned with organizational policies.

This approach ensures that each team has exactly the access they need and no more, following the principle of least privilege, which is a cornerstone of security in modern infrastructure.

