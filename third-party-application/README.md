## Monitor Third Party Application:

Kubernetes monitoring covers cluster components and Prometheus stack apps, but to monitor third-party apps like Redis at the application level (e.g., load, connections), we use a Prometheus exporter.

The exporter collects Redis metrics and exposes them at a `/metrics` endpoint. To let Prometheus scrape this data, we create a `ServiceMonitor` resource that points to the exporter.

---

## Deploy Redis Exporter:

We'll deploy a Redis exporter in Kubernetes using a Helm chart, allowing Prometheus to collect Redis application metrics (e.g. memory usage, connection count).

➡️ [Redis Exporter Helm Chart – Prometheus Community](https://github.com/prometheus-community/helm-charts/tree/main/charts/prometheus-redis-exporter)

The Redis exporter is deployed via the Prometheus Community Helm chart, with a custom `values.yaml` file to:

- Enable the ServiceMonitor
- Add a `release: monitoring` label so Prometheus detects the target
- Provide the correct Redis service address

Since Redis in this setup has no password, authentication isn’t needed. Optional alert rules can be added later via a separate `PrometheusRule` resource for flexibility. After deployment, Redis metrics appear in Prometheus under a new target.

### redis-values.yaml
```yaml
serviceMonitor:
  enabled: true
  labels:
    release: monitoring   # Ensure this matches the Prometheus setup
redisAddress: redis://redis-cart:6379  # Check that redis-cart is the correct service name for your Redis instance
````

### Commands:

```bash
kubectl get servicemonitor -n monitoring
# Print servicemonitor 

kubectl get servicemonitor monitoring-kube-prometheus-coredns -n monitoring -o yaml | less
# Check all servicemonitor components have labels, release: monitoring

kubectl get svc | grep -i redis
# Get Redis service name (e.g., redis-cart)

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
# Add Redis exporter Helm chart repo

helm repo update
# Update Helm repo

helm install redis-exporter prometheus-community/prometheus-redis-exporter -f redis-values.yaml
# Install Redis exporter using custom values file

helm ls
# List deployed Helm charts

kubectl get pods
# Check for deployed Redis exporter pod
# Example output:
# redis-exporter-prometheus-redis-exporter-68774fbc9-klhb8

kubectl get servicemonitor
# Check Redis exporter ServiceMonitor in default namespace
# Output:
# redis-exporter-prometheus-redis-exporter
```

---

## Alerting and Grafana Dashboard for Redis:

This section explains how to create Prometheus alert rules for a Redis application. It focuses on two key scenarios: when the Redis application is down (not accessible) and when it has an excessive number of connections.

The process involves creating a PrometheusRule YAML file (`redis-rules.yaml`) defining these alerts within the default namespace where Redis is running. Instead of manually writing the rules, it references the "Awesome Prometheus Alerts" documentation, which provides pre-written alert rules for various services, including Redis.

The example rules use metrics like `redis_up` to detect downtime (value of 0 is critical) and `redis_connected_clients` to identify too many connections (threshold of 100 is used as an example).

After applying this PrometheusRule using `kubectl apply -f redis-rules.yaml`, these new Redis-specific alert rules should become active in the Prometheus UI alongside existing alerts.

### Two key alerts are configured:

1. **Redis Down** – triggers when the Redis instance is unavailable.
2. **Too Many Connections** – triggers if Redis has over 100 connections, suggesting it's overloaded.

➡️ [Awesome Prometheus Alerts](https://awesome-prometheus-alerts.grep.to/)

### redis-rules.yaml

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: redis-rules
  labels:
    app: kube-prometheus-stack
    release: monitoring
spec: 
  groups:
  - name: redis.rules
    rules:
    - alert: RedisDown
      expr: redis_up == 0
      for: 0m
      labels:
        severity: critical
      annotations:
        summary: Redis down (instance {{ $labels.instance }})
        description: "Redis instance is down\n  VALUE = {{ $value }}\n  LABELS = {{ $labels }}"
    - alert: RedisTooManyConnections
      expr: redis_connected_clients / redis_config_maxclients * 100 > 90
      for: 2m
      labels:
        severity: warning
      annotations:
        summary: Redis too many connections (instance {{ $labels.instance }})
        description: "Redis is running out of connections (> 90% used)\n  VALUE = {{ $value }}\n  LABELS = {{ $labels }}"
```

### Commands:

```bash
kubectl apply -f redis-rules.yaml
# Apply the rules

kubectl get prometheusrule
# Verifying rule created. 'redis-rules' should be listed
```

---

## Trigger "Redis Down":

We'll trigger the "Redis down" alert. By scaling the Redis cart deployment down to zero replicas, the Redis pod is removed. After the Prometheus scrape interval (30s), the Redis exporter reports that the Redis application is unavailable (`redis_up == 0`), causing the alert to fire.

If Alertmanager is configured, a notification is sent. Scaling the deployment back up will resolve the alert.

### Commands:

```bash
kubectl get pod
# This is the Redis pod 'redis-cart-786fdb8948-lqtgv'

kubectl edit deployment redis-cart
# Edit the Redis deployment and set replicas to 0

kubectl get pod
# Redis cart pod is gone
```

---

## Create Redis Dashboard in Grafana:

Visualize alert data in Grafana for better analysis. Instead of creating a dashboard from scratch, use pre-configured dashboards from Grafana Labs.

Search for "Prometheus Redis Exporter Grafana dashboard" and ensure it matches the Redis exporter used in your setup. These dashboards provide panels for key metrics like memory usage and connected clients.

---

## Import Grafana Dashboard:

Importing existing Grafana dashboards is simple:

* In Grafana, go to **Create → Import**
* Paste the dashboard ID from Grafana Labs
* Select your Prometheus data source

Grafana will connect to the Redis exporter and display visualizations of relevant metrics like uptime, connected clients, etc.

### Commands:

```bash
kubectl get service
# Get Redis exporter service name: redis-exporter-prometheus-redis-exporter

kubectl describe service redis-exporter-prometheus-redis-exporter
# Check endpoint (e.g., 10.244.0.129:9121) used by both dashboard and node-exporter
```

---

## License

This project is licensed under the MIT License.

```text
MIT License
