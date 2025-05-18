# Monitor Third Party Application

### Topics of the Demo Project

#### Configure Monitoring for a Third-Party Application


Kubernetes monitoring covers cluster components and Prometheus stack apps, but to monitor third-party apps like Redis at the application level (e.g., load, connections), we use a Prometheus exporter.

The exporter collects Redis metrics and exposes them at a `/metrics` endpoint. To let Prometheus scrape this data, we create a `ServiceMonitor` resource that points to the exporter.

---

## Technologies Used

- Prometheus
- Kubernetes
- Redis
- Helm
- Grafana

---

## Project Description

Monitor Redis by using Prometheus Exporter:

- Deploy Redis service in our cluster
- Deploy Redis exporter using Helm Chart
- Configure Alert Rules (when Redis is down or has too many connections)
- Import Grafana Dashboard for Redis to visualize monitoring data in Grafana

---

## Deploy Redis Exporter

We'll deploy a Redis exporter in Kubernetes using a Helm chart, allowing Prometheus to collect Redis application metrics (e.g., memory usage, connection count).

### Steps to Deploy Redis Exporter Using Helm Chart

The most popular Redis exporter is available at [oliver006/redis_exporter](https://github.com/oliver006/redis_exporter). Since we want to deploy it in a Kubernetes cluster, we are going to use a Helm chart to do this.

Resources:

- [ArtifactHub](https://artifacthub.io/packages/helm/prometheus-community/prometheus-redis-exporter)
- [GitHub Helm Chart](https://github.com/prometheus-community/helm-charts/tree/main/charts/prometheus-redis-exporter)

In the [values.yaml](https://github.com/prometheus-community/helm-charts/blob/main/charts/prometheus-redis-exporter/values.yaml) file of the chart, you can see what variables may be set/overridden.

We need to know the name and port of the Redis service in the K8s cluster:

```sh
kubectl get services
# NAME                                       TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
# redis-exporter-prometheus-redis-exporter   ClusterIP      10.100.137.137   <none>        9121/TCP       43m
# rediscart                                  ClusterIP      10.100.206.38    <none>        6379/TCP       10d
# shippingservice                            ClusterIP      10.100.95.95     <none>        50051/TCP      10d

````

The service name is `rediscart` and it is listening on port `6379`.

Create a file `redis-values.yaml`:

```yaml
serviceMonitor:
  enabled: true
  labels:
    release: monitoring

redisAddress: redis://rediscart:6379
```

Deploy the Helm chart:

```sh
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
# "prometheus-community" already exists with the same configuration, skipping

helm repo update
# Hang tight while we grab the latest from your chart repositories...
# ...Successfully got an update from the "ingress" chart repository
# ...Successfully got an update from the "bitnami" chart repository
# ...Successfully got an update from the "prometheus-community" chart repository
# Update Complete. ⎈Happy Helming!⎈

helm install redis-exporter prometheus-community/prometheus-redis-exporter -f redis-values.yaml
# NAME: redis-exporter
# LAST DEPLOYED: Sat Sep 16 22:31:50 2023
# NAMESPACE: default
# STATUS: deployed
# REVISION: 1
# TEST SUITE: None
# NOTES:
# 1. Get the Redis Exporter URL by running these commands:
#   export POD_NAME=$(kubectl get pods --namespace default -l "app=prometheus-redis-exporter,release=redis-exporter" -o jsonpath="{.items[0].metadata.name}")
#   echo "Visit http://127.0.0.1:8080 to use your application"
#   kubectl port-forward $POD_NAME 8080:

helm ls
# NAME          	NAMESPACE	REVISION	UPDATED                              	STATUS  	CHART                          	APP VERSION
# redis-exporter	default  	1       	2023-09-16 22:31:50.514561 +0200 CEST	deployed	prometheus-redis-exporter-5.6.0	v1.54.0 

kubectl get pods
# NAME                                                       READY   STATUS    RESTARTS   AGE
# adservice-599678658f-2r7w2                                 1/1     Running   0          10d
# ...
# redis-exporter-prometheus-redis-exporter-6595c4bf5-phxgq   1/1     Running   0          2m26s
# rediscart-f5cdf4c67-q9sdk                                  1/1     Running   0          10d
# ...

kubectl get servicemonitors
# NAME                                       AGE
# redis-exporter-prometheus-redis-exporter   5m20s

kubectl get servicemonitor redis-exporter-prometheus-redis-exporter -o yaml
# apiVersion: monitoring.coreos.com/v1
# kind: ServiceMonitor
# metadata:
#   annotations:
#     meta.helm.sh/release-name: redis-exporter
#     meta.helm.sh/release-namespace: default
#   creationTimestamp: "2023-09-16T20:31:51Z"
#   generation: 1
#   labels:
#     app.kubernetes.io/managed-by: Helm
#     release: monitoring                              <---------------
#   name: redis-exporter-prometheus-redis-exporter
#   namespace: default
#   resourceVersion: "1985765"
#   uid: bbff5ba5-263f-4823-b067-59010f374a75
# spec:
#   endpoints:
#   - port: redis-exporter
#   jobLabel: redis-exporter-prometheus-redis-exporter
#   namespaceSelector:
#     matchNames:
#     - default
#   selector:
#     matchLabels:
#       app.kubernetes.io/instance: redis-exporter
#       app.kubernetes.io/name: prometheus-redis-exporter
```

Open the Prometheus UI in the browser and select Status > Targets. You should see a new target called 'serviceMonitor/default/redis-exporter-prometheus-redis-exporter/0 (1/1 up)' with a 'Last Scrape' time of something like '28.127s ago' which means that Prometheus detected the new metrics endpoint and successfully scraped it. If you type 'redis' into the query execution input field, you should see all the metrics starting with redis (make sure the 'Enable autocomplete' checkbox is checked), e.g. 'redis_connected_clients'.

---

## Alerting and Grafana Dashboard for Redis

This section explains how to create Prometheus alert rules for a Redis application. It focuses on two key scenarios: when the Redis application is down (not accessible) and when it has an excessive number of connections. The process involves creating a PrometheusRule YAML file (redis-rules.yaml) defining these alerts within the default namespace where Redis is running. Instead of manually writing the rules, it references the "Awesome Prometheus Alerts" documentation, which provides pre-written alert rules for various services, including Redis. The example rules use metrics like redis_up to detect downtime (value of 0 is critical) and redis_connected_clients to identify too many connections (threshold of 100 is used as an example). After applying this PrometheusRule using kubectl apply -f redis-rules.yaml, these new Redis-specific alert rules should become active in the Prometheus UI alongside existing alerts.

We will define two key alerts:

1. **Redis Down** – triggers when the Redis instance is unavailable.
2. **Too Many Connections** – triggers if Redis has over 90% of max connections used.

Reference: [Awesome Prometheus Alerts](https://awesome-prometheus-alerts.grep.to/)

Create the alert rule YAML file: `redis-rules.yaml`


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

Apply the alert rule:

```sh
kubectl apply -f redis-rules.yaml
##applying the rules

kubectl get prometheusrule
verifying rule created. 'redis-rules' created
```

---

## Trigger "Redis Down"

We'll trigger the "Redis down" alert. By scaling the Redis cart deployment down to zero replicas, the Redis pod is removed. After the Prometheus scrape interval (currently 30 seconds), the Redis exporter reports that the Redis application is unavailable (redis_up equals zero), causing the "Redis down" alert to fire in Prometheus. If Alertmanager is configured, a notification would be sent to the specified receiver. Finally, the Redis deployment is scaled back up to restore the Redis pod and resolve the alert, which should return to a green (non-firing) state after the next Prometheus scrape.

1. Identify the Redis pod:

```sh
kubectl get pod
# redis-cart-786fdb8948-lqtgv
```

2. Edit the deployment:

```sh
kubectl edit deployment redis-cart
# edit redis deployment "redis-cart"
# set replicas to 0
```

3. Check if pod is gone:

```sh
kubectl get pod
```

Prometheus should detect `redis_up == 0` and fire the alert.

---

## Create Redis Dashboard in Grafana

We'll manage visualizing alert data in Grafana for better analysis. Instead of creating a Redis dashboard from scratch, it recommends using pre-configured dashboards available on Grafana Labs. The process involves searching for relevant dashboards (e.g., "Prometheus Redis Exporter Grafana dashboard") and ensuring they utilize the same metrics as the Redis exporter being used in the cluster. The example highlights a suitable dashboard that provides panels for key Redis metrics like memory usage and connected clients, aiding in understanding the context of triggered alerts.

### Steps to Import Redis Dashboard in Grafana

1. Go to [Grafana Labs Dashboards](https://grafana.com/grafana/dashboards/).

2. Search for: `Redis Dashboard for Prometheus Redis Exporter 1.x`

3. Use Dashboard ID: **763**

4. In Grafana:

   * Go to `http://localhost:8080/dashboards`
   * Click "+" > Import
   * Enter **763** in "Import via grafana.com"
   * Click Load
   * Select Prometheus as the data source
   * Click Import

Verify data source matches the correct Redis exporter instance:

```sh
kubectl get services
# NAME                                       TYPE           CLUSTER-IP       EXTERNAL-IP                                                                 PORT(S)        AGE
# adservice                                  ClusterIP      10.100.22.140    <none>                                                                      9555/TCP       10d
# ...
# redis-exporter-prometheus-redis-exporter   ClusterIP      10.100.233.156   <none>                                                                      9121/TCP       13h
# rediscart                                  ClusterIP      10.100.206.38    <none>                                                                      6379/TCP       10d
# shippingservice                            ClusterIP      10.100.95.95     <none>                                                                      50051/TCP      10d

kubectl describe service redis-exporter-prometheus-redis-exporter
# Name:              redis-exporter-prometheus-redis-exporter
# Namespace:         default
# Labels:            app.kubernetes.io/instance=redis-exporter
#                    app.kubernetes.io/managed-by=Helm
#                    app.kubernetes.io/name=prometheus-redis-exporter
#                    app.kubernetes.io/version=v1.54.0
#                    helm.sh/chart=prometheus-redis-exporter-5.6.0
# Annotations:       meta.helm.sh/release-name: redis-exporter
#                    meta.helm.sh/release-namespace: default
# Selector:          app.kubernetes.io/instance=redis-exporter,app.kubernetes.io/name=prometheus-redis-exporter
# Type:              ClusterIP
# IP Family Policy:  SingleStack
# IP Families:       IPv4
# IP:                10.100.233.156
# IPs:               10.100.233.156
# Port:              redis-exporter  9121/TCP
# TargetPort:        exporter-port/TCP
# Endpoints:         192.168.1.149:9121           <---------------
# Session Affinity:  None
# Events:            <none>

kubectl describe service redis-exporter-prometheus-redis-exporter
# Endpoints: 192.168.1.149:9121
```

---
