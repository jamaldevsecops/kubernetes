MetalLB is a load-balancer implementation for Kubernetes clusters running on bare metal. It provides network load-balancing to external services in environments where cloud-based load-balancers are not available. Here are some key points about MetalLB:

1. **Modes**: MetalLB supports two modes of operation:
   - **Layer 2 (ARP/NDP)**: Suitable for small clusters or environments where you control the network.
   - **BGP**: Suitable for larger or more complex networks, allowing MetalLB to participate in BGP peering.

2. **IP Address Management**: MetalLB manages a pool of IP addresses and assigns them to Kubernetes services that require external access.

3. **Configuration**: MetalLB is configured using Kubernetes ConfigMaps and requires defining address pools and the operating mode.

4. **Installation**: MetalLB can be installed using Kubernetes manifests or Helm charts.

Here is a step-by-step guide to installing MetalLB using Kubernetes manifests.

### Step 1: Pre-requisite

 **Enable staticARP**:  
```sh
kubectl edit configmap -n kube-system kube-proxy
```
Change the value of `staticARP` from `false` to `true`
```
    ipvs:
      excludeCIDRs: null
      minSyncPeriod: 0s
      scheduler: ""
      strictARP: true
      syncPeriod: 0s
      tcpFinTimeout: 0s
      tcpTimeout: 0s
      udpTimeout: 0s
```

### Step 2: Install MetalLB

 **Apply the MetalLB manifests**:  
   source: https://metallb.io/installation/  
```sh
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.8/config/manifests/metallb-native.yaml
kubectl get pods -n metallb-system
```

### Step 3: Configure MetalLB

1. **Create a IPAddressPool and L2Advertisement for MetalLB configuration**:
    You need to define an address pool and mode of operation (Layer 2 or BGP).

    - **IPAddressPool Configuration**:
      Create a file named `IPAddressPool.yaml`:
      ```yaml
	   apiVersion: metallb.io/v1beta1
	   kind: IPAddressPool
	   metadata:
	     name: metallb-ip-pool
	     namespace: metallb-system
	   spec:
	     addresses:
	     - 192.168.20.131-192.168.20.135
      ```
      Make sure to replace the IP address range with a range suitable for your network.

    - **L2Advertisement Configuration**:
      Create a file named `L2Advertisement.yaml`:
      ```yaml
	   apiVersion: metallb.io/v1beta1
	   kind: L2Advertisement
	   metadata:
	     name: l2-advertisement
	     namespace: metallb-system
	   spec:
	     ipAddressPools:
	     - metallb-ip-pool
      ```

2. **Apply the IPAddressPool and L2Advertisement**:
    ```sh
    kubectl apply -f IPAddressPool.yaml
    kubectl apply -f L2Advertisement.yaml
    ```
    ```sh
    kubectl get ipaddresspools -n metallb-system
    kubectl get l2advertisements -n metallb-system
    ```


### Verification

1. **Check MetalLB pods**:
    ```sh
    kubectl get pods -n metallb-system
    ```
    You should see pods for `controller` and `speaker` running.

2. **Deploy a sample application with a LoadBalancer service**:
    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: nginx-service
      namespace: default
    spec:
      selector:
        app: nginx
      ports:
        - protocol: TCP
          port: 80
          targetPort: 80
      type: LoadBalancer
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: nginx-deployment
      namespace: default
    spec:
      selector:
        matchLabels:
          app: nginx
      replicas: 1
      template:
        metadata:
          labels:
            app: nginx
        spec:
          containers:
          - name: nginx
            image: nginx:latest
            ports:
            - containerPort: 80
    ```

3. **Apply the sample application**:
    ```sh
    kubectl apply -f nginx-deployment.yaml
    ```

4. **Check the external IP**:
    ```sh
    kubectl get svc nginx-service
    ```
    You should see an external IP assigned to the `nginx-service`.


Here is the step-by-step procedure to install MetalLB using Helm.

### Step 1: Install Helm

If you haven't installed Helm yet, you can follow these steps:

1. **Download Helm**:
    ```sh
    curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
    ```

2. **Install Helm**:
    ```sh
    chmod 700 get_helm.sh
    ./get_helm.sh
    ```

### Step 2: Add MetalLB Helm Repository

1. **Add the MetalLB repository**:
    ```sh
    helm repo add metallb https://metallb.github.io/metallb
    helm repo update
    ```

### Step 3: Install MetalLB

1. **Create the MetalLB namespace**:
    ```sh
    kubectl create namespace metallb-system
    ```

2. **Install MetalLB using Helm**:
    ```sh
    helm install metallb metallb/metallb -n metallb-system
    ```

### Step 4: Configure MetalLB

1. **Create a ConfigMap for MetalLB configuration**:
    You need to define an address pool and mode of operation (Layer 2 or BGP).

    - **Layer 2 Configuration**:
      Create a file named `metallb-config.yaml`:
      ```yaml
      apiVersion: v1
      kind: ConfigMap
      metadata:
        namespace: metallb-system
        name: config
      data:
        config: |
          address-pools:
          - name: default
            protocol: layer2
            addresses:
            - 192.168.1.240-192.168.1.250
      ```
      Make sure to replace the IP address range with a range suitable for your network.

2. **Apply the ConfigMap**:
    ```sh
    kubectl apply -f metallb-config.yaml
    ```

### Verification

1. **Check MetalLB pods**:
    ```sh
    kubectl get pods -n metallb-system
    ```
    You should see pods for `controller` and `speaker` running.

2. **Deploy a sample application with a LoadBalancer service**:
    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: nginx-service
      namespace: default
    spec:
      selector:
        app: nginx
      ports:
        - protocol: TCP
          port: 80
          targetPort: 80
      type: LoadBalancer
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: nginx-deployment
      namespace: default
    spec:
      selector:
        matchLabels:
          app: nginx
      replicas: 1
      template:
        metadata:
          labels:
            app: nginx
        spec:
          containers:
          - name: nginx
            image: nginx:latest
            ports:
            - containerPort: 80
    ```

3. **Apply the sample application**:
    ```sh
    kubectl apply -f nginx-deployment.yaml
    ```

4. **Check the external IP**:
    ```sh
    kubectl get svc nginx-service
    ```
    You should see an external IP assigned to the `nginx-service`.


You will access the Nginx web server using the external IP address assigned to the `nginx-service` by MetalLB.

Here are the steps to find the external IP and access the Nginx web server:

### Step 1: Check the External IP

1. **Get the service details**:
    ```sh
    kubectl get svc nginx-service
    ```

2. **Find the external IP**:
    The output will look something like this:
    ```
    NAME            TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)        AGE
    nginx-service   LoadBalancer   10.96.0.1       192.168.1.241    80:31952/TCP   2m
    ```

    In this example, `192.168.1.241` is the external IP assigned by MetalLB.

### Step 2: Access the Nginx Web Server

1. **Open a web browser**.

2. **Enter the external IP**:
    ```
    http://192.168.1.241
    ```

This should bring up the default Nginx welcome page if everything is set up correctly.





