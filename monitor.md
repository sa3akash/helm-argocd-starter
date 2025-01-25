To install the Prometheus stack in Kubernetes using Helm and ensure all data is visible in the dashboard, follow these steps:

---

### **Prerequisites**
1. **Kubernetes Cluster**: Ensure you have a running Kubernetes cluster and `kubectl` configured.
2. **Helm Installed**: Ensure Helm is installed and configured (`helm version` to verify).
3. **Namespace Setup**: Decide on a namespace (e.g., `monitoring`) for Prometheus installation.

---

### **Step 1: Add the Prometheus Community Helm Repository**
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

---

### **Step 2: Create a Namespace**
```bash
kubectl create namespace monitoring
```

---

### **Step 3: Install the Prometheus Stack**
Use the `kube-prometheus-stack` chart to install Prometheus and Grafana:
```bash
helm install prometheus-stack prometheus-community/kube-prometheus-stack --namespace monitoring
```

This will install:
- Prometheus (for metrics collection)
- Grafana (for dashboards)
- Alertmanager
- Node Exporter and more

---

### **Step 4: Access Grafana Dashboard**
#### **a) Retrieve Grafana Admin Password:**
```bash
kubectl get secret -n monitoring prometheus-stack-grafana -o jsonpath="{.data.admin-password}" | base64 --decode
```

#### **b) Port-Forward Grafana Service:**
```bash
kubectl port-forward -n monitoring svc/prometheus-stack-grafana 3000:80
```

Access Grafana at: [http://localhost:3000](http://localhost:3000)

Login with:
- **Username**: `admin`
- **Password**: (retrieved above)

---

### **Step 5: Verify Metrics in Prometheus**
#### **a) Access Prometheus:**
Port-forward Prometheus service:
```bash
kubectl port-forward -n monitoring svc/prometheus-stack-kube-prometheus-prometheus 9090
```
Access Prometheus at: [http://localhost:9090](http://localhost:9090)

#### **b) Verify Targets and Metrics:**
- Go to **Status > Targets** to ensure all targets are up.
- Go to **Graph** to query metrics (e.g., `node_cpu_seconds_total`).

---

### **Step 6: Configure Dashboards in Grafana**
1. Navigate to Grafana's **Dashboards** section.
2. Preconfigured dashboards should already be imported. For example:
    - Kubernetes/Nodes
    - Kubernetes/Pods
    - Kubernetes/Resources
3. If needed, import additional dashboards from the [Grafana Dashboards Library](https://grafana.com/grafana/dashboards/).

---

### **Step 7: Verify All Data in Dashboards**
1. Check Node metrics (CPU, memory, disk usage).
2. Ensure Kubernetes resources (pods, deployments) metrics are displayed.
3. Verify Grafana's data sources (should point to Prometheus).

---

### **Customizing Installation**
If you want custom settings (e.g., enabling persistent storage or alerting rules):
1. Download the default `values.yaml` file:
   ```bash
   helm show values prometheus-community/kube-prometheus-stack > values.yaml
   ```
2. Modify `values.yaml` as needed.
3. Install using the custom file:
   ```bash
   helm install prometheus-stack prometheus-community/kube-prometheus-stack --namespace monitoring -f values.yaml
   ```

---

### Troubleshooting Tips
1. **Missing Data**: Ensure node exporter, kube-state-metrics, and other components are running:
   ```bash
   kubectl get pods -n monitoring
   ```
2. **Grafana Dashboard Issues**: Validate data sources in Grafana under **Configuration > Data Sources**.

Let me know if you face any issues! 😊