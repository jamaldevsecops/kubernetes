To manage and switch between multiple AKS clusters across different Azure subscriptions, you can follow these steps:

### 1. Set Up Your Environment

Make sure you have the Azure CLI and `kubectl` installed and configured on your machine.

### 2. Configure Access to AKS Clusters

You need to log in to Azure, set the subscription context, and get credentials for each AKS cluster. Here's how you can do it for both clusters:

#### **For AKS Cluster 1:**

1. **Log in to Azure:**
   ```sh
   az login
   ```

   This command will prompt you to log in using your browser. Use the credentials for the appropriate Azure account.

2. **Set the subscription context:**
   ```sh
   az account set --subscription sdfsdfds-sdfsdfsdf-sfsdfdsf
   ```

3. **Get credentials for AKS Cluster 1:**
   ```sh
   az aks get-credentials --resource-group abc-RG --name abc-aks-k8s --context aks-cluster-1
   ```

   Here, `aks-cluster-1` is a context name you are assigning to the first cluster.

#### **For AKS Cluster 2:**

1. **Log in to Azure:**
   ```sh
   az login
   ```

   If you are using a different Azure account, make sure you log in with the credentials for the second account. You may need to log out first or use a different Azure CLI session.

2. **Set the subscription context:**
   ```sh
   az account set --subscription qwerwer-qewerwer0w-werwerwr
   ```

3. **Get credentials for AKS Cluster 2:**
   ```sh
   az aks get-credentials --resource-group xyz-RG --name xyz-aks-k8s --context aks-cluster-2
   ```

   Here, `aks-cluster-2` is a context name you are assigning to the second cluster.

### 3. List and Switch Contexts

After configuring the credentials for both clusters, you can list and switch between contexts using `kubectl`:

1. **List all contexts:**
   ```sh
   kubectl config get-contexts
   ```

   Example output:
   ```sh
   CURRENT   NAME            CLUSTER           AUTHINFO        NAMESPACE
   *         aks-cluster-1   abc-aks-k8s       user-abc-aks    default
             aks-cluster-2   xyz-aks-k8s       user-xyz-aks    default
   ```

2. **Switch to a different context:**
   ```sh
   kubectl config use-context aks-cluster-1
   kubectl config use-context aks-cluster-2
   ```

### Summary

- **Log in to Azure** and set the subscription context for each cluster.
- **Get credentials** for each AKS cluster and assign a unique context name.
- **List and switch** between contexts using `kubectl`.

This approach helps you manage multiple AKS clusters efficiently, even when they are in different subscriptions and have different Azure accounts.
