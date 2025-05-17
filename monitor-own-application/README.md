## Demo Project - Monitoring Own Applications

### Topics of the Demo Project
Configure Monitoring for Own Application

### Technologies Used
- Prometheus
- Kubernetes
- Node.js
- Grafana
- Docker
- Docker Hub

### Project Description
- Configure our NodeJS application to collect & expose Metrics with Prometheus Client Library 
- Deploy the NodeJS application, which has a metrics endpoint configured, into Kubernetes cluster 
- Configure Prometheus to scrape this exposed metrics and visualize it in Grafana Dashboard


#### Steps to configure our NodeJS application to collect & expose metrics
In order to monitor our own application we need to use the Prometheus client library within our application. There are client libraries for different programming languages like Java, Python, JavaScript (Node.js), etc. The provide an abstract interface to expose your metrics and implement the various Prometheus metric types like 'Counter', 'Gauge' 'Histogram' or 'Summary'.

The sample application we want to monitor is the Node.js application in the folder `node-project/app`.

We want to provide the following two metrics:
- the number of requests
- the duration of requests

A Prometheus client library for NodeJS applications can be found here: [https://www.npmjs.com/package/prom-client](https://www.npmjs.com/package/prom-client). So add the following dependency to the `package.json` file:
```json
"prom-client": "14.2.0"
```

In the `server.js` file replace the code
```js
...
let app = express();

app.get('/', function (req, res) {
  res.sendFile(path.join(__dirname, "index.html"));
});
...
```

with the following:
```js
...
let app = express();

const client = require('prom-client');
const collectDefaultMetrics = client.collectDefaultMetrics;
// probe every 5th second
collectDefaultMetrics({ timeout: 5000 });

const httpRequestsTotal = new client.Counter({
  name: 'http_request_operations_total',
  help: 'Total number of Http requests'
});

const httpRequestDurationSeconds = new client.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of Http requests in seconds',
  buckets: [0.1, 0.5, 2, 5, 10]
});

app.get('/metrics', async (req, res) => {
  res.set('Content-Type', client.register.contentType);
  res.end(await client.register.metrics());
});

app.get('/', function (req, res) {
  // simulate sleep for a random number of milliseconds
  var start = new Date();
  var simulateTime = Math.floor(Math.random() * (10000 - 500 + 1) + 500);

  setTimeout(function(argument) {
    // simulate execution time
    var end = new Date() - start;
    httpRequestDurationSeconds.observe(end / 1000); // convert to seconds
  }, simulateTime);

  httpRequestsTotal.inc();

  // do the real work
  res.sendFile(path.join(__dirname, "index.html"));
});
...
```

`Counter` is a cumulative metric, whose value can only increase. Resets to zero when the process restarts.

`Histogram` samples observations and counts them in configurable buckets. Tracks sizes and frequences of events.

So, as you see, as a developer you have to define the metrics and track the values in your application logic.

To test the metrics, execute the following commands:
```sh
cd node-project/app
npm install
npm start
# > bootcamp-node-project@1.0 start
# > node server.js
# 
# app listening on port 3000!
```

 Open the browser and navigate to 'http://localhost:3000'. Press the refresh button several times. Then navigate to 'http://localhost:3000/metrics'. You should see a bunch of default metrics and at the bottom our two custom metrics.

#### Steps to deploy the NodeJS application into Kubernetes cluster
To deploy the application in our K8s cluster we first have to make it available as a Docker image on our private DockerHub repository. Because we are building the Docker image on our local machine haaving an Apple M2 processor (arm64), but the image must run in the EKS cluster (amd64), we have to build the image using `buildx`:

```sh
cd node-project
docker buildx create --use
docker login
docker buildx build --platform linux/amd64 -t fsiegrist/fesi-repo:bootcamp-node-project-1.0.0 --push .
```

Now we can define Kubernetes configurations of a deployment and service for out node application. Create a file called `k8s-config.yaml` inside the node-project folder with the following content:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nodeapp
  labels:
    app: nodeapp
spec:
  selector:
    matchLabels:
      app: nodeapp
  template:
    metadata:
      labels:
        app: nodeapp
    spec:
      imagePullSecrets:
      - name: my-registry-key
      containers:
      - name: nodeapp
        image: fsiegrist/fesi-repo:bootcamp-node-project-1.0.0
        ports:
        - containerPort: 3000
        imagePullPolicy: Always  
---
apiVersion: v1
kind: Service
metadata:
  name: nodeapp
  labels:
    app: nodeapp
spec:
  type: ClusterIP
  selector:
    app: nodeapp
  ports:
  - name: service
    protocol: TCP
    port: 3000
    targetPort: 3000
```

To allow Kubernetes to pull the image from our local repository we have to provide the credentials in a Secret of type `docker-registry`. The secret is referenced from the deployment configuration using the name `my-registry-key` (see attribute 'imagePullSecrets'). So we have to name it like this. Create the Secret using the following command:

```sh
kubectl create secret docker-registry my-registry-key \
  --docker-server=docker.io \
  --docker-username=fsiegrist \
  --docker-password=<password>

# secret/my-registry-key created
```

Now we are ready to apply the deployment and service to the cluster:
```sh
kubectl apply -f node-project/k8s-config.yaml
# deployment.apps/nodeapp created
# service/nodeapp created

kubectl get services
# NAME       TYPE            CLUSTER-IP         EXTERNAL-IP        PORT(S)        AGE
# ...
# nodeapp    ClusterIP       10.100.77.236      <none>             3000/TCP       60s
# ...
```

To test the running application and the metrics do a port-forwarding...
```sh
kubectl port-forward svc/nodeapp 3000:3000
# Forwarding from 127.0.0.1:3000 -> 3000
# Forwarding from [::1]:3000 -> 3000
```
... and navigate to 'http://localhost:3000'. You should see the application running. Press the browser refresh button several times and then navigate to 'http://localhost:3000/metrics'. At the end of the metrics list you should see our two metrics:

```
# HELP http_request_operations_total Total number of Http requests
# TYPE http_request_operations_total counter
http_request_operations_total 11

# HELP http_request_duration_seconds Duration of Http requests in seconds
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{le="0.1"} 0
http_request_duration_seconds_bucket{le="0.5"} 0
http_request_duration_seconds_bucket{le="2"} 1
http_request_duration_seconds_bucket{le="5"} 3
http_request_duration_seconds_bucket{le="10"} 8
http_request_duration_seconds_bucket{le="+Inf"} 8
http_request_duration_seconds_sum 39.623
http_request_duration_seconds_count 8
```

#### Steps to configure Prometheus to scrape the metrics and visualize them in Grafana
To inform Prometheus about the new metrics endpoint of our application we have to add an according ServiceMonitor component to the K8s cluster.

Add the following configuration to the `k8s-config.yaml` file:
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: monitoring-node-app
  labels:
    release: monitoring
    app: nodeapp
spec:
  endpoints:
  - path: /metrics
    port: service
    targetPort: 3000
  namespaceSelector:
    matchNames:
    - default
  selector:
    matchLabels:
      app: nodeapp
```

Apply the new component to the cluster:
```sh
kubectl apply -f k8s-config.yaml
# servicemonitor.monitoring.coreos.com/monitoring-node-app created
```

Open the Prometheus UI (Status > Targets) and you should find the new target `serviceMonitor/default/monitoring-node-app/0 (0/1 up)`. At the beginning its state is 'UNKNOWN'. As soon as it got scraped for the first time, the state switches to 'UP'.

Navigate to the main page of the Prometheus UI and enter 'http_request' into the PromQL expression field. In the code completion suggests you should see the four new metrics of our application (all starting with http_request).

And on the Configuration page (Status > Configuration) the `scrape_configs` section contains a new job for our target:
```
scrape_configs:
- job_name: serviceMonitor/default/monitoring-node-app/0
  honor_timestamps: true
  scrape_interval: 30s
  scrape_timeout: 10s
  metrics_path: /metrics
  scheme: http
```

As a final step we want to create a Grafana dashboard to visualize our new metrics. Open Grafana UI, toggle the main menu in the top left corner, select 'Dashboards' and press 'New' and select 'New dashboard'. Press 'Add visualization' and select 'Prometheus' as the data source. Either create a new query with the query builder or directly type in the following query:\
`rate(http_request_operations_total[2m])`

Press 'Run queries to see the result of the query. Press 'Save' in the top right corner, enter 'Nodeapp Telemetry' as the name of the new dashboard and press 'Save'. Click on the three dots in the top right corner of the new panel, select 'Edit', click on the little `<` (Show options pane)on the top right (next to 'Time series') and enter 'Requests per secon' as the Panel Title. Press 'Apply' in the top right corner.

Let's add a second panel. Click 'Add' and select 'Visualization'. Enter 'Request Duration' as the panel title (in the options on the right), enter `rate(http_request_duration_seconds_sum[2m])` as the query, press 'Run queries' to check the result, press 'Save' and 'Apply' to save the changees and go back to the dashboard. You can rearrange the panels on the dashboard as you like by grabbing them on the top.

Also see the documentation of the [rate function](https://prometheus.io/docs/prometheus/latest/querying/functions/#rate).

