# Setup Prometheus Monitoring and Grafana Visualization in AWS EKS Cluster

Prometheus is a monitoring system that collects and stores metrics, supports alerting, and integrates with Grafana for advanced visualization.

---

## Topics of the Demo Project

- Install Prometheus Stack in Kubernetes

### Technologies Used

- Prometheus
- Kubernetes
- Helm
- AWS EKS
- eksctl
- Grafana
- Linux

---

## Project Description

- Setup EKS cluster using `eksctl`
- Deploy a microservices application
- Deploy Prometheus, Alert Manager, and Grafana in the cluster using the Prometheus Operator via Helm chart

---

## Steps to Setup EKS Cluster Using `eksctl`

Make sure your AWS credentials are configured in `~/.aws`.

```bash
# Create a cluster in the default region with two worker nodes
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



# After creation (approx. 10-15 mins), check nodes
kubectl get nodes

NAME                                             STATUS   ROLES    AGE     VERSION
ip-192-168-25-138.eu-central-1.compute.internal   Ready    <none>   5m45s   v1.25.12-eks-8ccc7ba
ip-192-168-86-219.eu-central-1.compute.internal   Ready    <none>   5m43s   v1.25.12-eks-8ccc7ba
```

---

## Deploy Microservices Application

Create a configuration file:

```bash
vim config-microservices.yaml
```

Paste the following:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: emailservice
spec:
  replicas: 2
  selector:
    matchLabels:
      app: emailservice
  template:
    metadata:
      labels:
        app: emailservice
    spec:
      containers:
      - name: server
        image: gcr.io/google-samples/microservices-demo/emailservice:v0.6.0
        ports:
        - containerPort: 8080
        env:
        - name: PORT
          value: "8080"
        readinessProbe:
          periodSeconds: 5
          exec:
            command: ["/bin/grpc_health_probe", "-addr=:8080"]
        livenessProbe:
          periodSeconds: 5
          exec:
            command: ["/bin/grpc_health_probe", "-addr=:8080"]
        resources:
          requests:
            cpu: 100m
            memory: 64Mi
          limits:
            cpu: 200m
            memory: 128Mi
---
apiVersion: v1
kind: Service
metadata:
  name: emailservice
spec:
  type: ClusterIP
  selector:
    app: emailservice
  ports:
  - protocol: TCP
    port: 5000
    targetPort: 8080
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: recommendationservice
spec:
  replicas: 2
  selector:
    matchLabels:
      app: recommendationservice
  template:
    metadata:
      labels:
        app: recommendationservice
    spec:
      containers:
      - name: server
        image: gcr.io/google-samples/microservices-demo/recommendationservice:v0.6.0
        ports:
        - containerPort: 8080
        env:
        - name: PORT
          value: "8080"
        - name: PRODUCT_CATALOG_SERVICE_ADDR
          value: "productcatalogservice:3550"
        readinessProbe:
          periodSeconds: 5
          exec:
            command: ["/bin/grpc_health_probe", "-addr=:8080"]
        livenessProbe:
          periodSeconds: 5
          exec:
            command: ["/bin/grpc_health_probe", "-addr=:8080"]
        resources:
          requests:
            cpu: 100m
            memory: 220Mi
          limits:
            cpu: 200m
            memory: 450Mi
---
apiVersion: v1
kind: Service
metadata:
  name: recommendationservice
spec:
  type: ClusterIP
  selector:
    app: recommendationservice
  ports:
  - protocol: TCP
    port: 8080
    targetPort: 8080
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: paymentservice
spec:
  replicas: 2
  selector:
    matchLabels:
      app: paymentservice
  template:
    metadata:
      labels:
        app: paymentservice
    spec:
      containers:
      - name: server
        image: gcr.io/google-samples/microservices-demo/paymentservice:v0.6.0
        ports:
        - containerPort: 50051
        env:
        - name: PORT
          value: "50051"
        - name: DISABLE_PROFILER
          value: "true"
        readinessProbe:
          periodSeconds: 5
          exec:
            command: ["/bin/grpc_health_probe", "-addr=:50051"]
        livenessProbe:
          periodSeconds: 5
          exec:
            command: ["/bin/grpc_health_probe", "-addr=:50051"]
        resources:
          requests:
            cpu: 100m
            memory: 64Mi
          limits:
            cpu: 200m
            memory: 128Mi
---
apiVersion: v1
kind: Service
metadata:
  name: paymentservice
spec:
  type: ClusterIP
  selector:
    app: paymentservice
  ports:
  - protocol: TCP
    port: 50051
    targetPort: 50051
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: productcatalogservice
spec:
  replicas: 2
  selector:
    matchLabels:
      app: productcatalogservice
  template:
    metadata:
      labels:
        app: productcatalogservice
    spec:
      containers:
      - name: server
        image: gcr.io/google-samples/microservices-demo/productcatalogservice:v0.6.0
        ports:
        - containerPort: 3550
        env:
        - name: PORT
          value: "3550"
        readinessProbe:
          periodSeconds: 5
          exec:
            command: ["/bin/grpc_health_probe", "-addr=:3550"]
        livenessProbe:
          periodSeconds: 5
          exec:
            command: ["/bin/grpc_health_probe", "-addr=:3550"]
        resources:
          requests:
            cpu: 100m
            memory: 64Mi
          limits:
            cpu: 200m
            memory: 128Mi
---
apiVersion: v1
kind: Service
metadata:
  name: productcatalogservice
spec:
  type: ClusterIP
  selector:
    app: productcatalogservice
  ports:
  - protocol: TCP
    port: 3550
    targetPort: 3550
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: currencyservice
spec:
  replicas: 2
  selector:
    matchLabels:
      app: currencyservice
  template:
    metadata:
      labels:
        app: currencyservice
    spec:
      containers:
      - name: server
        image: gcr.io/google-samples/microservices-demo/currencyservice:v0.6.0
        ports:
        - containerPort: 7000
        env:
        - name: PORT
          value: "7000"
        - name: DISABLE_PROFILER
          value: "true"
        readinessProbe:
          periodSeconds: 5
          exec:
            command: ["/bin/grpc_health_probe", "-addr=:7000"]
        livenessProbe:
          periodSeconds: 5
          exec:
            command: ["/bin/grpc_health_probe", "-addr=:7000"]
        resources:
          requests:
            cpu: 100m
            memory: 64Mi
          limits:
            cpu: 200m
            memory: 128Mi
---
apiVersion: v1
kind: Service
metadata:
  name: currencyservice
spec:
  type: ClusterIP
  selector:
    app: currencyservice
  ports:
  - protocol: TCP
    port: 7000
    targetPort: 7000
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shippingservice
spec:
  replicas: 2
  selector:
    matchLabels:
      app: shippingservice
  template:
    metadata:
      labels:
        app: shippingservice
    spec:
      containers:
      - name: server
        image: gcr.io/google-samples/microservices-demo/shippingservice:v0.6.0
        ports:
        - containerPort: 50051
        env:
        - name: PORT
          value: "50051"
        readinessProbe:
          periodSeconds: 5
          exec:
            command: ["/bin/grpc_health_probe", "-addr=:50051"]
        livenessProbe:
          periodSeconds: 5
          exec:
            command: ["/bin/grpc_health_probe", "-addr=:50051"]
        resources:
          requests:
            cpu: 100m
            memory: 64Mi
          limits:
            cpu: 200m
            memory: 128Mi
---
apiVersion: v1
kind: Service
metadata:
  name: shippingservice
spec:
  type: ClusterIP
  selector:
    app: shippingservice
  ports:
  - protocol: TCP
    port: 50051
    targetPort: 50051
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: adservice
spec:
  replicas: 2
  selector:
    matchLabels:
      app: adservice
  template:
    metadata:
      labels:
        app: adservice
    spec:
      containers:
      - name: server
        image: gcr.io/google-samples/microservices-demo/adservice:v0.6.0
        ports:
        - containerPort: 9555
        env:
        - name: PORT
          value: "9555"
        readinessProbe:
          initialDelaySeconds: 20
          periodSeconds: 15
          exec:
            command: ["/bin/grpc_health_probe", "-addr=:9555"]
        livenessProbe:
          initialDelaySeconds: 20
          periodSeconds: 15
          exec:
            command: ["/bin/grpc_health_probe", "-addr=:9555"]
        resources:
          requests:
            cpu: 200m
            memory: 180Mi
          limits:
            cpu: 300m
            memory: 300Mi
---
apiVersion: v1
kind: Service
metadata:
  name: adservice
spec:
  type: ClusterIP
  selector:
    app: adservice
  ports:
  - protocol: TCP
    port: 9555
    targetPort: 9555
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cartservice
spec:
  replicas: 2
  selector:
    matchLabels:
      app: cartservice
  template:
    metadata:
      labels:
        app: cartservice
    spec:
      containers:
      - name: server
        image: gcr.io/google-samples/microservices-demo/cartservice:v0.6.0
        ports:
        - containerPort: 7070
        env:
        - name: REDIS_ADDR
          value: "rediscart:6379"
        readinessProbe:
          periodSeconds: 5
          exec:
            command: ["/bin/grpc_health_probe", "-addr=:7070"]
        livenessProbe:
          periodSeconds: 5
          exec:
            command: ["/bin/grpc_health_probe", "-addr=:7070"]
        resources:
          requests:
            cpu: 200m
            memory: 64Mi
          limits:
            cpu: 300m
            memory: 128Mi
---
apiVersion: v1
kind: Service
metadata:
  name: cartservice
spec:
  type: ClusterIP
  selector:
    app: cartservice
  ports:
  - protocol: TCP
    port: 7070
    targetPort: 7070
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: checkoutservice
spec:
  replicas: 2
  selector:
    matchLabels:
      app: checkoutservice
  template:
    metadata:
      labels:
        app: checkoutservice
    spec:
      containers:
        - name: server
          image: gcr.io/google-samples/microservices-demo/checkoutservice:v0.6.0
          ports:
          - containerPort: 5050
          env:
          - name: PORT
            value: "5050"
          - name: PRODUCT_CATALOG_SERVICE_ADDR
            value: "productcatalogservice:3550"
          - name: SHIPPING_SERVICE_ADDR
            value: "shippingservice:50051"
          - name: PAYMENT_SERVICE_ADDR
            value: "paymentservice:50051"
          - name: EMAIL_SERVICE_ADDR
            value: "emailservice:5000"
          - name: CURRENCY_SERVICE_ADDR
            value: "currencyservice:7000"
          - name: CART_SERVICE_ADDR
            value: "cartservice:7070"
          readinessProbe:
            periodSeconds: 5
            exec:
              command: ["/bin/grpc_health_probe", "-addr=:5050"]
          livenessProbe:
            periodSeconds: 5
            exec:
              command: ["/bin/grpc_health_probe", "-addr=:5050"]
          resources:
            requests:
              cpu: 100m
              memory: 64Mi
            limits:
              cpu: 200m
              memory: 128Mi
---
apiVersion: v1
kind: Service
metadata:
  name: checkoutservice
spec:
  type: ClusterIP
  selector:
    app: checkoutservice
  ports:
  - protocol: TCP
    port: 5050
    targetPort: 5050
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: server
          image: gcr.io/google-samples/microservices-demo/frontend:v0.6.0
          ports:
          - containerPort: 8080
          env:
          - name: PORT
            value: "8080"
          - name: PRODUCT_CATALOG_SERVICE_ADDR
            value: "productcatalogservice:3550"
          - name: CURRENCY_SERVICE_ADDR
            value: "currencyservice:7000"
          - name: CART_SERVICE_ADDR
            value: "cartservice:7070"
          - name: RECOMMENDATION_SERVICE_ADDR
            value: "recommendationservice:8080"
          - name: SHIPPING_SERVICE_ADDR
            value: "shippingservice:50051"
          - name: CHECKOUT_SERVICE_ADDR
            value: "checkoutservice:5050"
          - name: AD_SERVICE_ADDR
            value: "adservice:9555"
          readinessProbe:
            initialDelaySeconds: 10
            httpGet:
              path: "/_healthz"
              port: 8080
              httpHeaders:
              - name: "Cookie"
                value: "shop_session-id=x-readiness-probe"
          livenessProbe:
            initialDelaySeconds: 10
            httpGet:
              path: "/_healthz"
              port: 8080
              httpHeaders:
              - name: "Cookie"
                value: "shop_session-id=x-liveness-probe"
          resources:
            requests:
              cpu: 100m
              memory: 64Mi
            limits:
              cpu: 200m
              memory: 128Mi
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  type: LoadBalancer
  selector:
    app: frontend
  ports:
  - name: http
    port: 80
    targetPort: 8080
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rediscart
spec:
  replicas: 1
  selector:
    matchLabels:
      app: rediscart
  template:
    metadata:
      labels:
        app: rediscart
    spec:
      containers:
      - name: redis
        image: redis:alpine
        ports:
        - containerPort: 6379
        readinessProbe:
          periodSeconds: 5
          tcpSocket:
            port: 6379
        livenessProbe:
          periodSeconds: 5
          tcpSocket:
            port: 6379
        resources:
          requests:
            cpu: 70m
            memory: 200Mi
          limits:
            cpu: 125m
            memory: 256Mi
        volumeMounts:
        - mountPath: /data
          name: redis-data
      volumes:
      - name: redis-data
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: rediscart
spec:
  type: ClusterIP
  selector:
    app: rediscart
  ports:
  - name: redis
    port: 6379
    targetPort: 6379    
```

Apply the configuration:

```bash
kubectl apply -f config-microservices.yaml

# deployment.apps/emailservice created
# service/emailservice created
# deployment.apps/recommendationservice created
# service/recommendationservice created
# deployment.apps/paymentservice created
# service/paymentservice created
# deployment.apps/productcatalogservice created
# service/productcatalogservice created
# deployment.apps/currencyservice created
# service/currencyservice created
# deployment.apps/shippingservice created
# service/shippingservice created
# deployment.apps/adservice created
# service/adservice created
# deployment.apps/cartservice created
# service/cartservice created
# deployment.apps/checkoutservice created
# service/checkoutservice created
# deployment.apps/frontend created
# service/frontend created
# deployment.apps/rediscart created
# service/rediscart created

kubectl get pods

# NAME                                     READY   STATUS    RESTARTS   AGE
# adservice-599678658f-2r7w2               1/1     Running   0          62s
# adservice-599678658f-p5vqq               1/1     Running   0          62s
# cartservice-7f96d4d6c9-fmhwp             1/1     Running   0          62s
# cartservice-7f96d4d6c9-m46ck             1/1     Running   0          62s
# checkoutservice-645cc49cd7-nq8jt         1/1     Running   0          62s
# checkoutservice-645cc49cd7-zftnc         1/1     Running   0          61s
# currencyservice-65d78749bb-4k7qg         1/1     Running   0          62s
# currencyservice-65d78749bb-bc9r5         1/1     Running   0          62s
# emailservice-556cd6f9b9-6cqhw            1/1     Running   0          63s
# emailservice-556cd6f9b9-jkw9b            1/1     Running   0          63s
# frontend-7dcff99cc6-68fsk                1/1     Running   0          61s
# frontend-7dcff99cc6-bw8tm                1/1     Running   0          61s
# paymentservice-695d749644-cwfs8          1/1     Running   0          63s
# paymentservice-695d749644-xl7qq          1/1     Running   0          63s
# productcatalogservice-8596c6c8cf-d896g   1/1     Running   0          62s
# productcatalogservice-8596c6c8cf-nzvtc   1/1     Running   0          62s
# recommendationservice-84f7fb567c-4c79j   1/1     Running   0          63s
# recommendationservice-84f7fb567c-g8dwx   1/1     Running   0          63s
# rediscart-f5cdf4c67-q9sdk                1/1     Running   0          61s
# shippingservice-59c76d497c-687kv         1/1     Running   0          62s
# shippingservice-59c76d497c-n9cr4         1/1     Running   0          62s

kubectl get services

# NAME                    TYPE           CLUSTER-IP       EXTERNAL-IP                                                                 PORT(S)        AGE
# adservice               ClusterIP      10.100.22.140    <none>                                                                      9555/TCP       104m
# cartservice             ClusterIP      10.100.206.3     <none>                                                                      7070/TCP       104m
# checkoutservice         ClusterIP      10.100.215.205   <none>                                                                      5050/TCP       104m
# currencyservice         ClusterIP      10.100.96.68     <none>                                                                      7000/TCP       104m
# emailservice            ClusterIP      10.100.175.35    <none>                                                                      5000/TCP       104m
# frontend                LoadBalancer   10.100.57.134    aef508a483cb942da8affab91f564595-502614928.eu-central-1.elb.amazonaws.com   80:31545/TCP   104m
# kubernetes              ClusterIP      10.100.0.1       <none>                                                                      443/TCP        117m
# paymentservice          ClusterIP      10.100.208.157   <none>                                                                      50051/TCP      104m
# productcatalogservice   ClusterIP      10.100.141.240   <none>                                                                      3550/TCP       104m
# recommendationservice   ClusterIP      10.100.57.197    <none>                                                                      8080/TCP       104m
# rediscart               ClusterIP      10.100.206.38    <none>                                                                      6379/TCP       104m
# shippingservice         ClusterIP      10.100.95.95     <none>                                                                      50051/TCP      104m
```

Access the application in browser:

```
http://uydwu677888cndnjd783y78yr4b7387f378-64bj26552.eu-west-2.elb.amazonaws.com
```

---

## Deploying Prometheus in Kubernetes

### Deployment Methods

1. Manual YAML files (complex)
2. Using Operator (simpler)
3. Using Helm Chart (recommended)

We’ll use the Helm chart method.

Helm Chart: [https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)

---

## Deploy Prometheus Stack Using Helm

```bash
# Add Helm repositories
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Create monitoring namespace
kubectl create namespace monitoring
kubectl get ns

# Install Prometheus stack
helm install monitoring prometheus-community/kube-prometheus-stack -n monitoring
```

Retrieve Grafana admin password:

```bash
kubectl --namespace monitoring get secrets monitoring-grafana -o jsonpath="{.data.admin-password}" | base64 -d ; echo
```

Port-forward Grafana:

```bash
export POD_NAME=$(kubectl --namespace monitoring get pod -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=monitoring" -o name)
kubectl --namespace monitoring port-forward $POD_NAME 3000
```

Get resources:

```bash
kubectl get all -n monitoring

# NAME                                                         READY   STATUS    RESTARTS   AGE
# pod/alertmanager-monitoring-kube-prometheus-alertmanager-0   2/2     Running   0          2m35s
# pod/monitoring-grafana-57b47fdd87-j762g                      3/3     Running   0          2m41s
# pod/monitoring-kube-prometheus-operator-7cc5877b9d-t2zdj     1/1     Running   0          2m41s
# pod/monitoring-kube-state-metrics-55b56ffff7-6m5mb           1/1     Running   0          2m41s
# pod/monitoring-prometheus-node-exporter-7hkvl                1/1     Running   0          2m41s
# pod/monitoring-prometheus-node-exporter-r8x8q                1/1     Running   0          2m41s
# pod/prometheus-monitoring-kube-prometheus-prometheus-0       2/2     Running   0          2m35s
# 
# NAME                                              TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
# service/alertmanager-operated                     ClusterIP   None             <none>        9093/TCP,9094/TCP,9094/UDP   2m35s
# service/monitoring-grafana                        ClusterIP   10.100.22.92     <none>        80/TCP                       2m41s
# service/monitoring-kube-prometheus-alertmanager   ClusterIP   10.100.214.211   <none>        9093/TCP,8080/TCP            2m41s
# service/monitoring-kube-prometheus-operator       ClusterIP   10.100.79.107    <none>        443/TCP                      2m41s
# service/monitoring-kube-prometheus-prometheus     ClusterIP   10.100.145.159   <none>        9090/TCP,8080/TCP            2m41s
# service/monitoring-kube-state-metrics             ClusterIP   10.100.117.112   <none>        8080/TCP                     2m41s
# service/monitoring-prometheus-node-exporter       ClusterIP   10.100.141.223   <none>        9100/TCP                     2m41s
# service/prometheus-operated                       ClusterIP   None             <none>        9090/TCP                     2m35s
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

```

```bash
kubectl --namespace monitoring get pods -l "release=monitoring"

# NAME                                                   READY   STATUS    RESTARTS   AGE
# monitoring-kube-prometheus-operator-7cc5877b9d-t2zdj   1/1     Running   0          81s
# monitoring-kube-state-metrics-55b56ffff7-6m5mb         1/1     Running   0          81s
# monitoring-prometheus-node-exporter-7hkvl              1/1     Running   0          81s
# monitoring-prometheus-node-exporter-r8x8q              1/1     Running   0          81s

```

---

## Prometheus Stack Components

### StatefulSets

* prometheus-monitoring-kube-prometheus-prometheus
* alertmanager-monitoring-kube-prometheus-alertmanager

### Deployments

* monitoring-grafana
* monitoring-kube-prometheus-operator
* monitoring-kube-state-metrics

### DaemonSet

* monitoring-prometheus-node-exporter

---

## Configuration Resources

### ConfigMaps

```bash
kubectl get configmap -n monitoring
```

### Secrets

```bash
kubectl get secret -n monitoring
```

### CRDs

```bash
kubectl get crds
```

---

## Prometheus UI Access

```bash
kubectl port-forward service/monitoring-kube-prometheus-prometheus -n monitoring 9090:9090
```

Visit: [http://127.0.0.1:9090](http://127.0.0.1:9090)

---

## Visualizing with Grafana

Grafana helps visualize metrics from Prometheus.

### Dashboard Structure

* Folders → Dashboards → Rows → Panels

Uses PromQL queries.

### Access Grafana

1. Get admin password:

```bash
kubectl --namespace monitoring get secrets monitoring-grafana -o jsonpath="{.data.admin-password}" | base64 -d
```

2. Port-forward:

```bash
kubectl port-forward deployment/monitoring-grafana -n monitoring 3000:3000
```

3. Visit Grafana:

```
http://localhost:3000
```

* Username: admin
* Password: prom-operator

---

## Optional: Explore Internals

### Grafana Deployment

```bash
kubectl get deployment -n monitoring
kubectl describe deployment monitoring-grafana -n monitoring > monitoring-grafana.yaml
```

### Operator Deployment

```bash
kubectl describe deployment monitoring-kube-prometheus-operator -n monitoring > operator-deployment.yaml
```

### Prometheus Secrets

```bash
kubectl get secret -n monitoring | grep prometheus-monitoring-kube-prometheus-prometheus
kubectl get secret prometheus-monitoring-kube-prometheus-prometheus -n monitoring -o yaml > config-reloader-volume-secret.yaml
```

---

## License

```
MIT License
```

---

## Summary

The Prometheus stack provides:

* Node-level monitoring with Node Exporter
* Cluster-level monitoring with Kube State Metrics
* Alerting with Alertmanager
* Visualization with Grafana
* Automated setup and management using the Prometheus Operator

```
