AKS Cluster Monitoring with Prometheus and Grafana on Azure VM

This guide documents the full process of setting up Prometheus and Grafana on an Azure Ubuntu VM (via Docker) to monitor an AKS cluster, and deploying a sample static HTML application on AKS to verify connectivity and LoadBalancer service.

1. VM Setup

Create an Ubuntu VM in Azure with a Public IP and allow inbound traffic on ports 22, 80, 3000, 9090 (or temporarily allow all ports for testing).

SSH into the VM:

ssh <user>@<vm_public_ip>


Update and install prerequisites:

sudo apt update
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
sudo snap install kubectl --classic
sudo apt-get update && sudo apt install docker.io -y
sudo systemctl enable docker --now
az login --use-device-code

2. Create and Connect to an AKS Cluster

If you don’t already have a cluster:

az aks create \
  --resource-group lucky \
  --name lucky-aks-cluster11 \
  --node-count 1 \
  --generate-ssh-keys

az aks get-credentials \
  --resource-group lucky \
  --name lucky-aks-cluster11


Verify:

kubectl get nodes

3. Run Prometheus in Docker
3.1 Create a Prometheus config file
sudo mkdir -p /opt/monitoring/prometheus
sudo tee /opt/monitoring/prometheus/prometheus.yml >/dev/null <<'YAML'
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
YAML


Check the file:

ls -l /opt/monitoring/prometheus/prometheus.yml
file /opt/monitoring/prometheus/prometheus.yml

3.2 Run Prometheus container
sudo docker rm -f prometheus 2>/dev/null || true
sudo docker run -d --name prometheus \
  -p 9090:9090 \
  -v /opt/monitoring/prometheus:/etc/prometheus:ro \
  -v prometheus-data:/prometheus \
  --restart unless-stopped \
  prom/prometheus:latest \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/prometheus \
  --web.enable-lifecycle


Now visit Prometheus at:
http://<VM_PUBLIC_IP>:9090

4. Run Grafana in Docker
sudo docker rm -f grafana 2>/dev/null || true
sudo docker run -d --name grafana \
  -p 3000:3000 \
  -e GF_SECURITY_ADMIN_USER=admin \
  -e GF_SECURITY_ADMIN_PASSWORD=admin123 \
  -v grafana-data:/var/lib/grafana \
  --restart unless-stopped \
  grafana/grafana:latest


Visit Grafana:
http://<VM_PUBLIC_IP>:3000
(Default login: admin / admin123)

5. Deploy a Sample HTML App on AKS

To verify your AKS cluster and networking, deploy a simple NGINX pod that serves a static HTML page.

Create the file:

vi html-demo.yaml


Paste:

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: html-files
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <head>
      <meta charset="utf-8"/>
      <title>AKS HTML Demo</title>
      <style>body{font-family:sans-serif;margin:40px} code{background:#eee;padding:2px 4px}</style>
    </head>
    <body>
      <h1>Hello from AKS</h1>
      <p>This is a static HTML page served by NGINX in AKS.</p>
      <p>Pod: <code>$(HOSTNAME)</code></p>
    </body>
    </html>
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: html-demo
  labels:
    app: html-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: html-demo
  template:
    metadata:
      labels:
        app: html-demo
    spec:
      containers:
        - name: nginx
          image: nginx:alpine
          ports:
            - containerPort: 80
          volumeMounts:
            - name: html
              mountPath: /usr/share/nginx/html
          resources:
            requests:
              cpu: "50m"
              memory: "64Mi"
            limits:
              cpu: "200m"
              memory: "256Mi"
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 15
            periodSeconds: 20
      volumes:
        - name: html
          configMap:
            name: html-files
---
apiVersion: v1
kind: Service
metadata:
  name: html-demo
  labels:
    app: html-demo
spec:
  type: LoadBalancer
  selector:
    app: html-demo
  ports:
    - port: 80
      targetPort: 80


Apply:

kubectl apply -f html-demo.yaml


Check service and wait for the external IP:

kubectl get svc html-demo --watch


Visit:
http://<EXTERNAL-IP>/

6. Deploy kube-prometheus-stack on AKS (via Helm)

Install Helm if not already installed:

curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash


Add Prometheus chart repo:

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update


Create a custom values file:

cat > values-kps.yaml <<'YAML'
grafana:
  enabled: false

kubeStateMetrics:
  enabled: true

nodeExporter:
  enabled: true

prometheus:
  service:
    type: ClusterIP
YAML


Install the stack:

helm install monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace -f values-kps.yaml


Check:

kubectl get pods -n monitoring
kubectl get svc -n monitoring

7. Access Cluster Prometheus from the VM

Forward Prometheus port from the cluster to the VM:

kubectl -n monitoring port-forward svc/monitoring-kube-prometheus-prometheus 9091:9090


Now you can scrape cluster metrics from http://localhost:9091.

Update the local Prometheus config to include this target:

sudo tee /opt/monitoring/prometheus/prometheus.yml >/dev/null <<'YAML'
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
  - job_name: 'aks-cluster'
    static_configs:
      - targets: ['host.docker.internal:9091']
YAML


Restart Prometheus:

sudo docker restart prometheus

8. Import Dashboards in Grafana

Open Grafana (http://<VM_PUBLIC_IP>:3000).

Add Prometheus as a data source (http://<VM_PUBLIC_IP>:9090).

Import recommended dashboards by ID:

Dashboard	ID
Kubernetes / Compute Resources / Cluster	315
Kubernetes / Compute Resources / Namespace	3146
Kubernetes / Pods	6417
Node Exporter Full	1860
Kubernetes API Server	12006
9. Verification

In Prometheus UI → Status > Targets → verify AKS exporters are UP.

In Grafana → dashboards show CPU, memory, node and pod metrics.
