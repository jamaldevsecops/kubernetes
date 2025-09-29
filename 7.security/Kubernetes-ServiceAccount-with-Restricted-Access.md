# Kubernetes ServiceAccount with Restricted Access

This guide explains how to create a Kubernetes ServiceAccount with limited permissions for running:
- `kubectl get pod`
- `kubectl exec -it deploy/<deployment_name> -c <container_name> -- bash`

It covers **both short-lived and long-lived tokens**.

---

## 🔹 Part 1: ServiceAccount, Role, and RoleBinding

Save this as `sa-role.yaml`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ngd-dev-user
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: ngd-dev-role
  namespace: default
rules:
# Allow get/list pods
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
# Allow exec into pods
- apiGroups: [""]
  resources: ["pods/exec"]
  verbs: ["create"]
# Allow reading pod logs
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ngd-dev-binding
  namespace: default
subjects:
- kind: ServiceAccount
  name: ngd-dev-user
  namespace: default
roleRef:
  kind: Role
  name: ngd-dev-role
  apiGroup: rbac.authorization.k8s.io
```

Apply it:
```bash
kubectl apply -f sa-role.yaml
```

---

## 🔹 Part 2: Short-Lived Token (Default)

1. Generate token (valid ~1 hour):
   ```bash
   kubectl -n default create token ngd-dev-user
   ```

2. Get cluster details:
   ```bash
   kubectl config view --raw -o jsonpath='{.clusters[0].cluster.server}'
   kubectl config view --raw -o jsonpath='{.clusters[0].cluster.certificate-authority-data}'
   ```

3. Build `~/.kube/config`:
   ```yaml
   apiVersion: v1
   kind: Config
   clusters:
   - cluster:
       certificate-authority-data: <CA_FROM_STEP_2>
       server: <SERVER_FROM_STEP_2>
     name: ngd-dev-cluster
   contexts:
   - context:
       cluster: ngd-dev-cluster
       namespace: default
       user: ngd-dev-user
     name: ngd-dev-context
   current-context: ngd-dev-context
   users:
   - name: ngd-dev-user
     user:
       token: <TOKEN_FROM_STEP_1>
   ```

---

## 🔹 Part 3: Long-Lived Token

1. Create a token secret `sa-token.yaml`:

   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: ngd-dev-user-token
     namespace: default
     annotations:
       kubernetes.io/service-account.name: ngd-dev-user
   type: kubernetes.io/service-account-token
   ```

   Apply it:
   ```bash
   kubectl apply -f sa-token.yaml
   ```

2. Get the permanent token:
   ```bash
   kubectl get secret ngd-dev-user-token -n default -o jsonpath='{.data.token}' | base64 -d
   ```

3. Get cluster details:
   ```bash
   kubectl config view --raw -o jsonpath='{.clusters[0].cluster.server}'
   kubectl config view --raw -o jsonpath='{.clusters[0].cluster.certificate-authority-data}'
   ```

4. Build `~/.kube/config`:
   ```yaml
   apiVersion: v1
   kind: Config
   clusters:
   - cluster:
       certificate-authority-data: <CA_FROM_STEP_3>
       server: <SERVER_FROM_STEP_3>
     name: ngd-dev-cluster
   contexts:
   - context:
       cluster: ngd-dev-cluster
       namespace: default
       user: ngd-dev-user
     name: ngd-dev-context
   current-context: ngd-dev-context
   users:
   - name: ngd-dev-user
     user:
       token: <TOKEN_FROM_STEP_2>
   ```

---

## 🔹 Testing Access

```bash
kubectl get pods
kubectl exec -it deploy/<deployment_name> -c <container_name> -- bash
```

✅ Allowed  
❌ Denied for other actions like delete/apply.

---

## Notes
- Short-lived tokens are valid for ~1 hour.  
- Long-lived tokens remain valid until the secret is deleted.  
- Keep tokens safe — they are equivalent to a password.

---

## 🔹 Ready-to-Use Kubeconfig Template (Long-Lived Token)

Save this as `ngd-dev-user.kubeconfig` (or in `~/.kube/config`):

```yaml
apiVersion: v1
kind: Config
clusters:
- cluster:
    certificate-authority-data: <BASE64-ENCODED-CA-HERE>
    server: <K8S-API-SERVER-URL-HERE>
  name: ngd-dev-cluster
contexts:
- context:
    cluster: ngd-dev-cluster
    namespace: default
    user: ngd-dev-user
  name: ngd-dev-context
current-context: ngd-dev-context
users:
- name: ngd-dev-user
  user:
    token: <LONG-LIVED-TOKEN-HERE>
```

### 🔹 How to use

1. Replace the placeholders:

| Placeholder                     | What to fill                                                   |
|---------------------------------|----------------------------------------------------------------|
| `<BASE64-ENCODED-CA-HERE>`      | Run: `kubectl config view --raw -o jsonpath='{.clusters[0].cluster.certificate-authority-data}'` |
| `<K8S-API-SERVER-URL-HERE>`     | Run: `kubectl config view --raw -o jsonpath='{.clusters[0].cluster.server}'` |
| `<LONG-LIVED-TOKEN-HERE>`       | Extract token from long-lived secret: `kubectl get secret ngd-dev-user-token -n default -o jsonpath='{.data.token}' | base64 -d` |

2. Save the file to `~/.kube/config` **or** keep it separate and use it via:
```bash
export KUBECONFIG=/path/to/ngd-dev-user.kubeconfig
```

3. Test access:
```bash
kubectl get pods
kubectl exec -it deploy/<deployment_name> -c <container_name> -- bash
```

- ✅ Allowed: `get pods`, `exec`  
- ❌ Denied: other actions (like delete, apply)
