
## Prometheus Installation
To install Prometheus in a Kubernetes cluster, you can either deploy Prometheus manually using Kubernetes manifests or use Helm, which is a more common and automated approach.

Here’s how you can install Prometheus using Helm:

Step 1: Install Helm (if not installed)

If Helm is not already installed, you can install it by following these commands.

# For Linux or MacOS
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# For Windows, use Chocolatey
choco install kubernetes-helm

Step 2: Add Prometheus Helm Repo

Prometheus can be installed from the Helm repository. First, add the Prometheus chart repository.

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

Step 3: Create a Namespace for Prometheus

It’s a good practice to create a separate namespace for Prometheus. You can do this using:

kubectl create namespace monitoring

Step 4: Install Prometheus

Now, you can install Prometheus by running the following Helm command.

helm install prometheus prometheus-community/prometheus --namespace monitoring

This will install Prometheus using the default configuration. It will deploy the following components:

	•	Prometheus server
	•	Alertmanager
	•	Node Exporter
	•	Pushgateway

You can check the status of the installation using:

kubectl get pods -n monitoring

Step 5: Accessing the Prometheus UI

By default, Prometheus is accessible only within the cluster (ClusterIP service). To access it externally, you can either:

	1.	Use port-forwarding, or
	2.	Expose Prometheus using an Ingress.

Option 1: Port-forwarding

kubectl --namespace monitoring port-forward svc/prometheus-server 9090:80

Now you can access the Prometheus UI by navigating to http://localhost:9090 in your browser.

Option 2: Expose via Ingress (Optional)

If you have an Ingress controller configured (e.g., Nginx), you can expose Prometheus externally by creating an Ingress resource.

cat > prometheus-ingress.yml << EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: prometheus-ingress
  namespace: monitoring
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: prometheus.yourdomain.com  # Replace with your domain
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: prometheus-server
            port:
              number: 80
EOF

kubectl apply -f prometheus-ingress.yml

Ensure that your DNS points to the cluster where Prometheus is deployed, and you should be able to access it externally.

Step 6: Customize Prometheus Configuration (Optional)

If you need to customize Prometheus (e.g., to scrape additional targets or integrate with Grafana), you can modify the values.yaml file of the Helm chart and reapply the installation.

To fetch the default values for Prometheus, run:

helm show values prometheus-community/prometheus > prometheus-values.yaml

Edit the prometheus-values.yaml file and apply it during installation:

helm install prometheus prometheus-community/prometheus --namespace monitoring -f prometheus-values.yaml

Step 7: Verify Installation

To ensure everything is working, you can access the Prometheus targets:

	•	Open the Prometheus UI at http://localhost:9090 or http://prometheus.yourdomain.com (if using Ingress).
	•	Go to Status > Targets to verify that Prometheus is scraping metrics from different sources.

This completes the basic installation of Prometheus on your Kubernetes cluster. If you’re planning to integrate it with Grafana for visualization, you can follow similar steps to install Grafana and set up Prometheus as a data source.