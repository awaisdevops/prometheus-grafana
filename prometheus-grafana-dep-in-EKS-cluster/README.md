# Setup Prometheus Monitoring and Grafana Visualization in AWS EKS Cluster

Prometheus is a monitoring system that collects and stores metrics, supports alerting, and integrates with Grafana for advanced visualization.

> **Note:**  
> This guide assumes you already have an AWS EKS cluster up and running.

---

## Deploying Prometheus in Kubernetes

There are three main approaches to deploy Prometheus in a Kubernetes cluster:

### 1. Manual Deployment
Involves writing and applying YAMLs for each component. It's complex and error-prone.

### 2. Using an Operator
Simplifies management by handling Prometheus components as a single unit.

### 3. Using a Helm Chart (Recommended)
The easiest and most efficient way. Helm installs the Prometheus Operator, which manages and automates the entire setup.

> We'll use the **Helm chart method** for deploying Prometheus.

---

## Deploy Prometheus Stack Using Helm

Deploy the Prometheus stack into its own namespace for better separation.

Helm Chart Repository:  
➡ [kube-prometheus-stack Helm Chart](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)

### Commands:

```bash
# Add Helm repositories
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add stable https://kubernetes-charts.storage.googleapis.com/
helm repo update

# Create a namespace for monitoring
kubectl create namespace monitoring

# Verify namespace creation
kubectl get ns

# Install the Prometheus stack
helm install monitoring prometheus-community/kube-prometheus-stack -n monitoring

# Retrieve Grafana admin password
kubectl --namespace monitoring get secrets monitoring-grafana -o jsonpath="{.data.admin-password}" | base64 -d ; echo

# Port-forward Grafana service
export POD_NAME=$(kubectl --namespace monitoring get pod -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=monitoring" -o name)
kubectl --namespace monitoring port-forward $POD_NAME 3000

# Get all resources in monitoring namespace
kubectl get all -n monitoring
```

---

## Understanding Prometheus Stack Components

### StatefulSets

- `prometheus-prometheus-kube-prometheus-prometheus`: Core Prometheus server managed by the operator.
- `alertmanager-prometheus-kube-prometheus-alertmanager`: Manages alerts.

### Deployments

- `prometheus-grafana`: Grafana visualization tool.
- `prometheus-kube-prometheus-operator`: Manages the stack.
- `prometheus-kube-state-metrics`: Collects Kubernetes state metrics.

### ReplicaSets

- Automatically created by Deployments (e.g., Grafana, kube-state-metrics).

### DaemonSet

- `prometheus-prometheus-node-exporter`: Collects node-level metrics like CPU, memory, and disk.

---

## ⚙️ Configuration Resources

### ConfigMaps

```bash
kubectl get configmap -n monitoring
```

Holds configuration files for Prometheus, Alertmanager, etc.

### Secrets

```bash
kubectl get secret -n monitoring
```

Stores sensitive data such as Grafana admin credentials.

### CRDs (Custom Resource Definitions)

```bash
kubectl get crds
```

Manages Prometheus resources like `ServiceMonitor`, `PrometheusRule`, etc.

---

## Summary

The Prometheus stack offers:

- **Node-level monitoring** with node-exporter.
- **Cluster component monitoring** with kube-state-metrics.
- **Alerting** with Alertmanager.
- **Visualization** with Grafana.
- Automated management via the **Prometheus Operator**.

---

## Exploring Prometheus Stack Components in Kubernetes

### StatefulSets

Inspect StatefulSets:

```bash
kubectl get statefulset -n monitoring
```

Expected:

- `prometheus-monitoring-kube-prometheus-prometheus`
- `alertmanager-monitoring-kube-prometheus-alertmanager`

### Inspect StatefulSets in Detail

```bash
kubectl describe statefulset prometheus-monitoring-kube-prometheus-prometheus -n monitoring > prom.yaml
kubectl describe statefulset alertmanager-monitoring-kube-prometheus-alertmanager -n monitoring > alert.yaml
```

Inside Prometheus pods:

- `prometheus`: Main server
- `config-reloader`: Sidecar container for dynamic config updates.

---

### Config Reloader Secrets

```bash
kubectl get secret -n monitoring | grep prometheus-monitoring-kube-prometheus-prometheus
kubectl get secret prometheus-monitoring-kube-prometheus-prometheus -n monitoring -o yaml > config-reloader-volume-secret.yaml
```

---

### Inspect Prometheus Operator

```bash
kubectl describe deployment monitoring-kube-prometheus-operator -n monitoring > operator-deployment.yaml
```

The operator handles:

- StatefulSets
- ConfigMaps and Secrets
- CRDs like `ServiceMonitor` and `PrometheusRule`

---

### Configuration Files

```bash
kubectl get configmap -n monitoring
```

---

## Prometheus UI Overview

Access Prometheus UI:

```bash
kubectl port-forward service/monitoring-kube-prometheus-prometheus -n monitoring 9090:9090
```

Visit: [http://127.0.0.1:9090](http://127.0.0.1:9090)

---

### Understanding Jobs and Instances

- A **job** is a logical grouping of scrape targets.
- An **instance** is a specific endpoint to scrape metrics from.

Example: `kubernetes-apiservers` job groups all API server pods.

---

## Visualizing Your EKS Cluster with Grafana

Grafana helps visualize the metrics collected by Prometheus.

---

### Grafana Dashboard Structure

- **Folders** ➔ **Dashboards** ➔ **Rows** ➔ **Panels**

Grafana executes **PromQL queries** against Prometheus.

---

### Accessing Grafana

1. Get Grafana admin password:

   ```bash
   kubectl --namespace monitoring get secrets monitoring-grafana -o jsonpath="{.data.admin-password}" | base64 -d
   ```

2. Port-forward Grafana service:

   ```bash
   kubectl port-forward deployment/monitoring-grafana -n monitoring 3000:3000
   ```

3. Open Grafana:

   ```
   http://localhost:3000
   ```
   
   - Username: `admin`
   - Password: `prom-operator`

---

### Optional: Inspect Grafana Deployment

```bash
kubectl get deployment -n monitoring
kubectl describe deployment monitoring-grafana -n monitoring > monitoring-grafana.yaml
kubectl get pods -n monitoring
kubectl logs <grafana-pod-name> -c grafana -n monitoring
```

---

### Manage Users and Data Sources

- **Users:** Add users via Grafana settings.
- **Data Sources:** Prometheus is pre-configured when using the Helm chart.

---

## License

```
MIT License
---
