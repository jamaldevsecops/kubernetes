
# NGINX Ingress Controller Installation with Helm
![image](https://github.com/user-attachments/assets/ffeb6db9-9e92-4b5f-aff8-7d0a2ee6880d)

This guide provides step-by-step instructions for installing the NGINX Ingress Controller using Helm on on-premises Kubernetes with MetalLB, AKS, and EKS.

## Prerequisites
- Kubernetes Cluster: A running Kubernetes cluster (version 1.19 or later recommended).
- Helm 3: Installed and configured (helm version --short should show v3.x.x).
- kubectl: Configured to interact with your cluster (kubectl cluster-info).
- MetalLB (for on-premises): Configured with an IP address pool.
- RBAC: Enabled on the cluster (if not, adjust --set rbac.create=false in Helm commands).
- Network Access: For on-premises, ensure ports 80 and 443 are open; for AKS/EKS, ensure cloud provider permissions for load balancers.

## 1. On-Premises Kubernetes with MetalLB
MetalLB provides load-balancer functionality for bare-metal clusters, assigning external IPs to services. Ensure MetalLB is installed and configured with an IP address pool (e.g., 192.168.1.100-192.168.1.200).
Steps

1. Add the NGINX Ingress Helm Repository:
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

2. Create a Namespace:
```bash
kubectl create namespace ingress-nginx
```

3. Install NGINX Ingress Controller:Use Helm to install the controller, configuring it to use MetalLB’s LoadBalancer service.
```bash
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --set controller.replicaCount=2 \
  --set controller.nodeSelector."kubernetes\.io/os"=linux \
  --set defaultBackend.nodeSelector."kubernetes\.io/os"=linux \
  --set controller.service.externalTrafficPolicy=Local
```
  - replicaCount=2: Ensures high availability with two replicas.
  - nodeSelector: Restricts to Linux nodes.
  - externalTrafficPolicy=Local: Preserves client source IP.

4. Verify the Installation:Check that the controller pods are running:
```bash
kubectl get pods -n ingress-nginx
```
Verify the LoadBalancer service has an external IP from MetalLB’s pool:
```bash
kubectl get svc -n ingress-nginx
```
Example output:
```plain
NAME                             TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)                      AGE
ingress-nginx-controller         LoadBalancer   10.96.123.45   192.168.1.100   80:31000/TCP,443:31001/TCP   5m
```

5. Test Access:Use the external IP (e.g., 192.168.1.100) to access the controller:
```bash
curl http://192.168.1.100
```
Expect a 404 response from NGINX (default behavior without Ingress resources).


### Notes
- Ensure MetalLB’s IP pool is dedicated and not overlapping with other network resources.
- If the external IP is <pending>, check MetalLB logs (kubectl logs -n metallb-system -l app=metallb).
- For HTTPS, configure TLS in your Ingress resources later.

## 2. Azure Kubernetes Service (AKS)
AKS integrates with Azure Load Balancer for external access. You may need an Azure static public IP for consistent addressing.

### Steps
1. Get the AKS Resource Group:Identify the node resource group for your AKS cluster:
```bash
az aks show --resource-group <AKS_RESOURCE_GROUP> --name <AKS_CLUSTER_NAME> --query nodeResourceGroup -o tsv
```
Example output: MC_aks-rg1_aksdemo1_westus

2. Create a Static Public IP (optional for fixed IP):
```bash
az network public-ip create \
  --resource-group <NODE_RESOURCE_GROUP> \
  --name myAKSPublicIPForIngress \
  --sku Standard \
  --allocation-method static \
  --query publicIp.ipAddress -o tsv
```
Note the IP (e.g., 52.154.156.139).

3. Add the NGINX Ingress Helm Repository:
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

4. Create a Namespace:
```bash
kubectl create namespace ingress-basic
```

5. Install NGINX Ingress Controller:
```bash
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-basic \
  --set controller.replicaCount=2 \
  --set controller.nodeSelector."kubernetes\.io/os"=linux \
  --set defaultBackend.nodeSelector."kubernetes\.io/os"=linux \
  --set controller.service.externalTrafficPolicy=Local \
  --set controller.service.loadBalancerIP="52.154.156.139"
```
  - Replace 52.154.156.139 with your static IP or omit for dynamic IP.
  - Use --set rbac.create=false if RBAC is disabled.


6. Verify the Installation:Check pods:
```bash
kubectl get pods -n ingress-basic
```
Verify the LoadBalancer service:
```bash
kubectl get svc -n ingress-basic
```
Example output:
```plain
NAME                             TYPE           CLUSTER-IP     EXTERNAL-IP       PORT(S)                      AGE
ingress-nginx-controller         LoadBalancer   10.0.123.45    52.154.156.139    80:31000/TCP,443:31001/TCP   5m
```

7. Test Access:
```bash
curl http://52.154.156.139
```
Expect a 404 response.


### Notes
- If using an internal load balancer, add --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-internal"=true.
- Check Azure portal for Load Balancer status under the node resource group.
- For TLS, integrate with Azure Key Vault or cert-manager later.

## 3. Amazon Elastic Kubernetes Service (EKS)
EKS uses AWS Network Load Balancer (NLB) or Classic Load Balancer. Ensure your IAM role has permissions for ELB.

### Steps
1. Add the NGINX Ingress Helm Repository:
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

2. Create a Namespace:
```bash
kubectl create namespace ingress-nginx
```

3. Install NGINX Ingress Controller with NLB:
```bash
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --set controller.replicaCount=2 \
  --set controller.nodeSelector."kubernetes\.io/os"=linux \
  --set defaultBackend.nodeSelector."kubernetes\.io/os"=linux \
  --set controller.service.externalTrafficPolicy=Local \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-type"="nlb"
```
- aws-load-balancer-type=nlb: Uses AWS NLB for better performance.


4. Verify the Installation:Check pods:
```bash
kubectl get pods -n ingress-nginx
```
Verify the LoadBalancer service:
```bash
kubectl get svc -n ingress-nginx
```
Example output:
```bash
NAME                             TYPE           CLUSTER-IP     EXTERNAL-IP                              PORT(S)                      AGE
ingress-nginx-controller         LoadBalancer   10.100.20.84   a4217761afdb3457683821e38a3d3de7.elb.us-east-2.amazonaws.com   80:31105/TCP,443:31746/TCP   5m
```

5. Test Access:Get the NLB DNS name and test:
```
curl http://a4217761afdb3457683821e38a3d3de7.elb.us-east-2.amazonaws.com
```
Expect a 404 response.

### Notes
- For Classic Load Balancer, omit the aws-load-balancer-type annotation.
- Update DNS records to point to the NLB DNS name.
- For SSL, use cert-manager with Let’s Encrypt.
- Ensure IAM roles allow ELB creation (AWSLoadBalancerControllerIAMPolicy).

## Post-Installation
- Ingress Resources: Create Ingress objects to route traffic to your services (e.g., apiVersion: networking.k8s.io/v1, kind: Ingress).
- TLS: Configure TLS with cert-manager or manual certificates.
- Monitoring: Enable Prometheus metrics with --set controller.metrics.enabled=true.
- Uninstall (if needed):helm uninstall ingress-nginx --namespace <NAMESPACE>
- kubectl delete namespace <NAMESPACE>
```bash
helm uninstall ingress-nginx --namespace <NAMESPACE>
kubectl delete namespace <NAMESPACE>
```

## Troubleshooting
- Pods Not Running: Check logs (kubectl logs -n <NAMESPACE> <POD_NAME>).
- No External IP: Verify MetalLB (on-premises) or cloud load balancer provisioning.
- 404 Errors: Ensure Ingress resources are correctly defined.
- RBAC Issues: Set --set rbac.create=false if RBAC is disabled.

