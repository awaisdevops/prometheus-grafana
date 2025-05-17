# Setup Prometheus Monitoring and Grafana Visualization in AWS EKS Cluster

Prometheus is a monitoring system that collects and stores metrics, supports alerting, and integrates with Grafana for advanced visualization.
---

Topics of the Demo Project
--------------------------

Install Prometheus Stack in Kubernetes
Technologies Used

    Prometheus
    Kubernetes
    Helm
    AWS EKS
    eksctl
    Grafana
    Linux

Project Description

    Setup EKS cluster using eksctl
    Deploy a micreoservices application
    Deploy Prometheus, Alert Manager and Grafana in cluster as part of the Prometheus Operator using Helm chart
---

Steps to setup EKS cluster using eksctl
---------------------------------------

Execute the following commands (make sure the AWS configuration and credentials in ~/.aws are set correctly):

'
# create a cluster in the default region with two worker nodes
eksctl create cluster

#2023-09-06 22:03:02 [ℹ]  eksctl version 0.141.0
#2023-09-06 22:03:02 [ℹ]  using region eu-central-1
#...
#2023-09-06 22:03:02 [ℹ]  building cluster stack "eksctl-ferocious-painting-1694030582-cluster"
#...
#2023-09-06 22:14:06 [ℹ]  building managed nodegroup stack "eksctl-ferocious-painting-1694030582-nodegroup-ng-1c743942"
#...
#2023-09-06 22:16:35 [✔]  saved kubeconfig as "/Users/fsiegrist/.kube/config"
#2023-09-06 22:16:35 [ℹ]  no tasks
#2023-09-06 22:16:35 [✔]  all EKS cluster resources for "ferocious-painting-1694030582" have been created
#2023-09-06 22:16:35 [ℹ]  nodegroup "ng-1c743942" has 2 node(s)
#2023-09-06 22:16:35 [ℹ]  node "ip-192-168-25-138.eu-central-1.compute.internal" is ready
#2023-09-06 22:16:35 [ℹ]  node "ip-192-168-86-219.eu-central-1.compute.internal" is ready
#2023-09-06 22:16:35 [ℹ]  waiting for at least 2 node(s) to become ready in "ng-1c743942"
#2023-09-06 22:16:35 [ℹ]  nodegroup "ng-1c743942" has 2 node(s)
#2023-09-06 22:16:35 [ℹ]  node "ip-192-168-25-138.eu-central-1.compute.internal" is ready
#2023-09-06 22:16:35 [ℹ]  node "ip-192-168-86-219.eu-central-1.compute.internal" is ready
#2023-09-06 22:16:36 [ℹ]  kubectl command should work with "/Users/fsiegrist/.kube/config", try 'kubectl get nodes'
#2023-09-06 22:16:36 [✔]  EKS cluster "ferocious-painting-1694030582" in "eu-central-1" region is ready


# when the cluster has been created (after about 10-15 minutes)
kubectl get nodes

# NAME                                              STATUS   ROLES    AGE     VERSION
# ip-192-168-25-138.eu-central-1.compute.internal   Ready    <none>   5m45s   v1.25.12-eks-8ccc7ba
# ip-192-168-86-219.eu-central-1.compute.internal   Ready    <none>   5m43s   v1.25.12-eks-8ccc7ba
'
---

Steps to deploy a Microservices Application
-------------------------------------------

Create the config file for all the required componenets:

-->> vim config-micorservices.yaml

'
apiVersion: apps/v1 #apiVersion #apps #v1
kind: Deployment #kind #Deployment
metadata:
  name: emailservice #metadata #name #emailservice
spec:
  replicas: 2 #spec #replicas #2
  selector:
    matchLabels:
      app: emailservice #selector #matchLabels #app #emailservice
  template:
    metadata:
      labels:
        app: emailservice #template #metadata #labels #app #emailservice
    spec:
      containers:
      - name: server #containers #name #server
        image: gcr.io/google-samples/microservices-demo/emailservice:v0.6.0 #image #containerImage #emailservice #v0.6.0
        ports:
        - containerPort: 8080 #ports #containerPort #8080
        env:
        - name: PORT #env #name #PORT
          value: "8080" #env #value #8080
        readinessProbe: #readinessProbe
          periodSeconds: 5 #readinessProbe #periodSeconds #5
          exec:
            command: ["/bin/grpc_health_probe", "-addr=:8080"] #readinessProbe #exec #grpc_health_probe #command
        livenessProbe: #livenessProbe
          periodSeconds: 5 #livenessProbe #periodSeconds #5
          exec:
            command: ["/bin/grpc_health_probe", "-addr=:8080"] #livenessProbe #exec #grpc_health_probe #command
        resources: #resources
          requests:
            cpu: 100m #resources #requests #cpu #100m
            memory: 64Mi #resources #requests #memory #64Mi
          limits:
            cpu: 200m #resources #limits #cpu #200m
            memory: 128Mi #resources #limits #memory #128Mi
---
apiVersion: v1 #apiVersion #v1
kind: Service #kind #Service
metadata:
  name: emailservice #metadata #name #emailservice
spec:
  type: ClusterIP #spec #type #ClusterIP
  selector:
    app: emailservice #selector #app #emailservice
  ports:
  - protocol: TCP #ports #protocol #TCP
    port: 5000 #ports #port #5000
    targetPort: 8080 #ports #targetPort #8080
---
apiVersion: apps/v1 #apiVersion #apps #v1
kind: Deployment #kind #Deployment
metadata:
  name: recommendationservice #metadata #name #recommendationservice
spec:
  replicas: 2 #spec #replicas #2
  selector:
    matchLabels:
      app: recommendationservice #selector #matchLabels #app #recommendationservice
  template:
    metadata:
      labels:
        app: recommendationservice #template #metadata #labels #app #recommendationservice
    spec:
      containers:
      - name: server #containers #name #server
        image: gcr.io/google-samples/microservices-demo/recommendationservice:v0.6.0 #image #containerImage #recommendationservice #v0.6.0
        ports:
        - containerPort: 8080 #ports #containerPort #8080
        env:
        - name: PORT #env #name #PORT
          value: "8080" #env #value #8080
        - name: PRODUCT_CATALOG_SERVICE_ADDR #env #name #PRODUCT_CATALOG_SERVICE_ADDR
          value: "productcatalogservice:3550" #env #value #productcatalogservice #3550
        readinessProbe: #readinessProbe
          periodSeconds: 5 #readinessProbe #periodSeconds #5
          exec:
            command: ["/bin/grpc_health_probe", "-addr=:8080"] #readinessProbe #exec #grpc_health_probe #command
        livenessProbe: #livenessProbe
          periodSeconds: 5 #livenessProbe #periodSeconds #5
          exec:
            command: ["/bin/grpc_health_probe", "-addr=:8080"] #livenessProbe #exec #grpc_health_probe #command
        resources: #resources
          requests:
            cpu: 100m #resources #requests #cpu #100m
            memory: 220Mi #resources #requests #memory #220Mi
          limits:
            cpu: 200m #resources #limits #cpu #200m
            memory: 450Mi #resources #limits #memory #450Mi
---
apiVersion: v1 #apiVersion #v1
kind: Service #kind #Service
metadata:
  name: recommendationservice #metadata #name #recommendationservice
spec:
  type: ClusterIP #spec #type #ClusterIP
  selector:
    app: recommendationservice #selector #app #recommendationservice
  ports:
  - protocol: TCP #ports #protocol #TCP
    port: 8080 #ports #port #8080
    targetPort: 8080 #ports #targetPort #8080
---
apiVersion: apps/v1 #apiVersion #apps #v1
kind: Deployment #kind #Deployment
metadata:
  name: paymentservice #metadata #name #paymentservice
spec:
  replicas: 2 #spec #replicas #2
  selector:
    matchLabels:
      app: paymentservice #selector #matchLabels #app #paymentservice
  template:
    metadata:
      labels:
        app: paymentservice #template #metadata #labels #app #paymentservice
    spec:
      containers:
      - name: server #containers #name #server
        image: gcr.io/google-samples/microservices-demo/paymentservice:v0.6.0 #image #containerImage #paymentservice #v0.6.0
        ports:
        - containerPort: 50051 #ports #containerPort #50051
        env:
        - name: PORT #env #name #PORT
          value: "50051" #env #value #50051
        - name: DISABLE_PROFILER #env #name #DISABLE_PROFILER
          value: "true" #env #value #true
        readinessProbe: #readinessProbe
          periodSeconds: 5 #readinessProbe #periodSeconds #5
          exec:
            command: ["/bin/grpc_health_probe", "-addr=:50051"] #readinessProbe #exec #grpc_health_probe #command
        livenessProbe: #livenessProbe
          periodSeconds: 5 #livenessProbe #periodSeconds #5
          exec:
            command: ["/bin/grpc_health_probe", "-addr=:50051"] #livenessProbe #exec #grpc_health_probe #command
        resources: #resources
          requests:
            cpu: 100m #resources #requests #cpu #100m
            memory: 64Mi #resources #requests #memory #64Mi
          limits:
            cpu: 200m #resources #limits #cpu #200m
            memory: 128Mi #resources #limits #memory #128Mi
---
apiVersion: v1 #apiVersion #v1
kind: Service #kind #Service
metadata:
  name: paymentservice #metadata #name #paymentservice
spec:
  type: ClusterIP #spec #type #ClusterIP
  selector:
    app: paymentservice #selector #app #paymentservice
  ports:
  - protocol: TCP #ports #protocol #TCP
    port: 50051 #ports #port #50051
    targetPort: 50051 #ports #targetPort #50051
---
apiVersion: apps/v1 #apiVersion #apps #v1
kind: Deployment #kind #Deployment
metadata:
  name: productcatalogservice #metadata #name #productcatalogservice
spec:
  replicas: 2 #spec #replicas #2
  selector:
    matchLabels:
      app: productcatalogservice #selector #matchLabels #app #productcatalogservice
  template:
    metadata:
      labels:
        app: productcatalogservice #template #metadata #labels #app #productcatalogservice
    spec:
      containers:
      - name: server #containers #name #server
        image: gcr.io/google-samples/microservices-demo/productcatalogservice:v0.6.0 #image #containerImage #productcatalogservice #v0.6.0
        ports:
        - containerPort: 3550 #ports #containerPort #3550
        env:
        - name: PORT #env #name #PORT
          value: "3550" #env #value #3550
        readinessProbe: #readinessProbe
          periodSeconds: 5 #readinessProbe #periodSeconds #5
          exec:
            command: ["/bin/grpc_health_probe", "-addr=:3550"] #readinessProbe #exec #grpc_health_probe #command
        livenessProbe: #livenessProbe
          periodSeconds: 5 #livenessProbe #periodSeconds #5
          exec:
            command: ["/bin/grpc_health_probe", "-addr=:3550"] #livenessProbe #exec #grpc_health_probe #command
        resources: #resources
          requests:
            cpu: 100m #resources #requests #cpu #100m
            memory: 64Mi #resources #requests #memory #64Mi
          limits:
            cpu: 200m #resources #limits #cpu #200m
            memory: 128Mi #resources #limits #memory #128Mi
---
apiVersion: v1 #apiVersion #v1
kind: Service #kind #Service
metadata:
  name: productcatalogservice #metadata #name #productcatalogservice
spec:
  type: ClusterIP #spec #type #ClusterIP
  selector:
    app: productcatalogservice #selector #app #productcatalogservice
  ports:
  - protocol: TCP #ports #protocol #TCP
    port: 3550 #ports #port #3550
    targetPort: 3550 #ports #targetPort #3550
---
apiVersion: apps/v1 #apiVersion #apps #v1
kind: Deployment #kind #Deployment
metadata:
  name: currencyservice #metadata #name #currencyservice
spec:
  replicas: 2 #spec #replicas #2
  selector:
    matchLabels:
      app: currencyservice #selector #matchLabels #app #currencyservice
  template:
    metadata:
      labels:
        app: currencyservice #template #metadata #labels #app #currencyservice
    spec:
      containers:
      - name: server #containers #name #server
        image: gcr.io/google-samples/microservices-demo/currencyservice:v0.6.0 #image #containerImage #currencyservice #v0.6.0
        ports:
        - containerPort: 7000 #ports #containerPort #7000
        env:
        - name: PORT #env #name #PORT
          value: "7000" #env #value #7000
        - name: DISABLE_PROFILER #env #name #DISABLE_PROFILER
          value: "true" #env #value #true
        readinessProbe: #readinessProbe
          periodSeconds: 5 #readinessProbe #periodSeconds #5
          exec:
            command: ["/bin/grpc_health_probe", "-addr=:7000"] #readinessProbe #exec #grpc_health_probe #command
        livenessProbe: #livenessProbe
          periodSeconds: 5 #livenessProbe #periodSeconds #5
          exec:
            command: ["/bin/grpc_health_probe", "-addr=:7000"] #livenessProbe #exec #grpc_health_probe #command
        resources: #resources
          requests:
            cpu: 100m #resources #requests #cpu #100m
            memory: 64Mi #resources #requests #memory #64Mi
          limits:
            cpu: 200m #resources #limits #cpu #200m
            memory: 128Mi #resources #limits #memory #128Mi
---
apiVersion: v1 #apiVersion #v1
kind: Service #kind #Service
metadata:
  name: currencyservice #metadata #name #currencyservice
spec:
  type: ClusterIP #spec #type #ClusterIP
  selector:
    app: currencyservice #selector #app #currencyservice
  ports:
  - protocol: TCP #ports #protocol #TCP
    port: 7000 #ports

---

kubectl apply -f config-microservices.yaml
# deployment.apps/emailservice created
# service/emailservice created


kubectl get pods
# NAME                                     READY   STATUS    RESTARTS   AGE
# adservice-599678658f-2r7w2               1/1     Running   0          62s
# adservice-599678658f-p5vqq               1/1     Running   0          62s

kubectl get services
# NAME                    TYPE           CLUSTER-IP       EXTERNAL-IP                                                                 PORT(S)        AGE
# adservice               ClusterIP      10.100.22.140    <none>                                                                      9555/TCP       104m
---

Open the browser and navigate to 'http://uydwu677888cndnjd783y78yr4b7387f378-64bj26552.eu-west-2.elb.amazonaws.com' to see the running application.
---

## Deploying Prometheus in Kubernetes
-------------------------------------

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
-------------------------------------

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

kubectl --namespace monitoring get pods -l "release=monitoring"
# NAME                                                   READY   STATUS    RESTARTS   AGE
# monitoring-kube-prometheus-operator-7cc5877b9d-t2zdj   1/1     Running   0          81s

# or
kubectl get all -n monitoring
# NAME                                                         READY   STATUS    RESTARTS   AGE
# pod/alertmanager-monitoring-kube-prometheus-alertmanager-0   2/2     Running   0          2m35s
# pod/monitoring-grafana-57b47fdd87-j762g                      3/3     Running   0          2m41s

# 
# NAME                                              TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
# service/alertmanager-operated                     ClusterIP   None             <none>        9093/TCP,9094/TCP,9094/UDP   2m35s
# service/monitoring-grafana                        ClusterIP   10.100.22.92     <none>        80/TCP                       2m41s

# 
# NAME                                                 DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
# daemonset.apps/monitoring-prometheus-node-exporter   2         2         2       2            2           kubernetes.io/os=linux   2m41s
# 
# NAME                                                  READY   UP-TO-DATE   AVAILABLE   AGE
# deployment.apps/monitoring-grafana                    1/1     1            1           2m41s
# deployment.apps/monitoring-kube-prometheus-operator   1/1     1            1           2m41s
# deployment.apps/monitoring-kube-state-metrics         1/1     1            1           2m41s
# 
# NAME                                                             DESIRED   CURRENT   READY   AGE
# replicaset.apps/monitoring-grafana-57b47fdd87                    1         1         1       2m42s
# replicaset.apps/monitoring-kube-prometheus-operator-7cc5877b9d   1         1         1       2m42s
# replicaset.apps/monitoring-kube-state-metrics-55b56ffff7         1         1         1       2m42s
# 
# NAME                                                                    READY   AGE
# statefulset.apps/alertmanager-monitoring-kube-prometheus-alertmanager   1/1     2m36s
# statefulset.apps/prometheus-monitoring-kube-prometheus-prometheus       1/1     2m36s

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
