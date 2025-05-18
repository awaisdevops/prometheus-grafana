# AlertManager Configuration

### Topics of the Demo Project
#Configure Alerting for our Application

When an alert rule in Prometheus goes into a "firing" state, it means Prometheus has detected that the defined condition (like high CPU load) has been met. Prometheus then sends this alert to Alertmanager. However, just because an alert is firing in Prometheus doesn't automatically mean you'll get a notification. Alertmanager is the component responsible for actually routing and sending out notifications (like emails). In the scenario described, Prometheus fired an alert, but since Alertmanager wasn't configured, the alert was essentially ignored, and no notification was sent. We'll configure Alertmanager to handle these firing alerts and send out email notifications.

---

### Technologies Used
- Prometheus
- Kubernetes
- Linux

### Project Description
Configure our Monitoring Stack to notify us whenever CPU usage exceeds 50% or a Pod cannot start.
- Configure Alert Rules in Prometheus Server
- Configure Alertmanager with Email Receiver

## Alertmanager Configuration File

Alertmanager is deployed by the Prometheus operator as an independent/dedicated application like we deployed Prometheus. So Prometheus is one application, Alertmanager is another application. And that means Alertmanager has its own configuration configured through a configuration file.

Alertmanager is a separate application from Prometheus, configured via its own file. Its basic UI allows viewing the configuration and filtering alerts. The configuration has three main sections:
- `global`: for overarching settings
- `routes`: to direct alerts to specific receivers based on matching labels (like `alertname`)
- `receivers`: which define the notification channels (e.g., email, Slack)

Currently, a "null" receiver is configured, causing all incoming alerts from Prometheus to be ignored. The `routes` section uses `match` attributes to specify which alerts go to which receivers. Additionally, there are grouping configurations to bundle alerts for more efficient notifications. The goal is to modify this configuration to add an email receiver and create routes for specific alerts like **"HostHighCPULoad"** and **"KubernetesPodCrashLooping"** to be sent via email.


## Managing Alertmanager Configuration in Kubernetes

The Alertmanager configuration is stored as a base64-encoded secret in Kubernetes, managed by the Alertmanager operator. While you can view the current configuration by decoding this secret, **direct editing isn't advised**. Instead, the recommended method for configuring Alertmanager is by using the **AlertmanagerConfig** custom Kubernetes resource provided by the monitoring API.

### Inspect Configuration

```bash
kubectl get pod -n monitoring
# Identify the Alertmanager Pod

kubectl describe pod alertmanager-monitoring-kube-prometheus-alertmanager-0 -n monitoring > alertmanager-pod.yaml
# Describe the Alertmanager Pod

# Analyze the Pod Description (alertmanager-pod.yaml):
# Check "args" for --config-file path (/etc/alertmanager/config/alertmanager.yaml.gz)
# Check "mounts" for /etc/alertmanager/config from config-volume (ro)
# Check "volumes" for config-volume source (secret: alertmanager-monitoring-kube-prometheus-alertmanager-generated)

kubectl get secret alertmanager-monitoring-kube-prometheus-alertmanager-generated -o yaml -n monitoring > alertmanager-config-from-secret1.yaml
kubectl get secret alertmanager-monitoring-kube-prometheus-alertmanager-generated -o yaml -n monitoring | less
# Retrieve the Alertmanager Secret

echo -n 'ENCODED_CONTENT_HERE' | base64 -d | less
# Attempt to Decode the Secret
```

> ⚠️ Alertmanager configurations are managed by the operator; direct secret editing is not recommended.

Use the AlertmanagerConfig object from the monitoring API to manage configuration.
Refer to: [AlertmanagerConfig CRD](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/monitoring_apis/alertmanagerconfig-monitoring-coreos-com-v1beta1)

---

#### Steps to configure Alert Rules in Prometheus Server
Create a file called `alert-rules.yaml` and add the first alert rule definition (CPU usage exceeds 50%):
```yaml
name: HostHighCpuLoad
expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[2m])) * 100) > 50
for: 2m
labels:
  severity: warning
  namespace: monitoring
annotations:
  description: "CPU load on host is over 50%\n Value = {{ $value }}\n Instance = {{ $label.instance }}\n"
  summary: Host CPU load high.
```

To develop the final value of the expression (`expr`), you can open the Prometheus UI, go to the main / home page ('http://localhost:9090/graph') and execute various expressions. For the example above you could test the following versions:
- `node_cpu_seconds_total{mode="idle"}`
- `rate(node_cpu_seconds_total{mode="idle"}[2m])`
- `(rate(node_cpu_seconds_total{mode="idle"}[2m])) * 100`
- `avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[2m])) * 100`
- `100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[2m])) * 100)`

Now we want to add the new alert rule to the set of already configured rules. To find out, where the rules are stored, open the Prometheus UI and select Status > Configuration. The `rule_files` section looks like this:
```yaml
rule_files:
- /etc/prometheus/rules/prometheus-monitoring-kube-prometheus-prometheus-rulefiles-0/*.yaml
```

Fortunately we don't have to find the ConfigMap holding these rule files and adjust it (add our new rule to it). Since we have deployed Prometheus using the Prometheus Operator, there is a much simpler way. The Operator lets us define custom Kubernetes components (specified by CRDs), which contain alert rules. The operator will then tell Prometheus to reload its configuration.

To define a custom K8s component for alert rules, adjust the content of the `alert-rules.yaml` file as follows:
```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: main-rules
  namespace: monitoring
  labels: # needed for the operator to find the components that need a configuration reload
    app: kube-prometheus-stack
    release: monitoring
spec:
  groups:
    - name: main.rules
      rules:
      - alert: HostHighCpuLoad
        expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[2m])) * 100) > 50
        for: 2m
        labels:
          severity: warning
          namespace: monitoring  # <-- a matcher for this label is added when the AlertmanagerConfig is applied and reloaded
        annotations:
          description: "CPU load on host is over 50%\n Value = {{ $value }}\n Instance = {{ $labels.instance }}\n"
          summary: Host CPU load high.
```

To find out the structure of the `spec` attribute, which is specific for each K8s resource, check the [documentation](https://docs.openshift.com/container-platform/4.13/rest_api/monitoring_apis/prometheusrule-monitoring-coreos-com-v1.html) of the custom component `PrometheusRule` (to find the documentation, do a Google search for 'PrometheusRule monitoring.coreos.com').

Now we add the second rule (Pod cannot start) to the file:
```yaml
      - alert: KubernetesPodCrashLooping
        expr: kube_pod_container_status_restarts_total > 5
        for: 0m
        labels:
          severity: critical
          namespace: monitoring
        annotations:
          description: "Pod {{ $labels.pod }} is crash looping\n Value = {{ $value }}"
          summary: Kubernetes pod crash looping.
```

Now we can apply the new component specification to the K8s cluster:
```sh
kubectl apply -f alert-rules.yaml
# prometheusrule.monitoring.coreos.com/main-rules created

kubectl get PrometheusRules -n monitoring
# NAME                                                              AGE
# main-rules                                                        54s  <--
# monitoring-kube-prometheus-alertmanager.rules                     6d
# monitoring-kube-prometheus-config-reloaders                       6d
# monitoring-kube-prometheus-etcd                                   6d
# monitoring-kube-prometheus-general.rules                          6d
# monitoring-kube-prometheus-k8s.rules                              6d
# ...
```

To find out whether the rules-configuration has been reloaded, we can inspect the logs of the according container:
```sh
kubectl get pods -n monitoring
# NAME                                                     READY   STATUS    RESTARTS   AGE
# alertmanager-monitoring-kube-prometheus-alertmanager-0   2/2     Running   0          6d
# monitoring-grafana-57b47fdd87-j762g                      3/3     Running   0          6d
# monitoring-kube-prometheus-operator-7cc5877b9d-t2zdj     1/1     Running   0          6d
# monitoring-kube-state-metrics-55b56ffff7-6m5mb           1/1     Running   0          6d
# monitoring-prometheus-node-exporter-7hkvl                1/1     Running   0          6d
# monitoring-prometheus-node-exporter-r8x8q                1/1     Running   0          6d
# prometheus-monitoring-kube-prometheus-prometheus-0       2/2     Running   0          6d

kubectl logs prometheus-monitoring-kube-prometheus-prometheus-0 -n monitoring -c config-reloader
# level=info ts=2023-09-06T20:31:51.222379258Z caller=main.go:115 msg="Starting prometheus-config-reloader" version="(version=0.67.1, branch=refs/tags/v0.67.1, revision=ce782eafad7fb29bd4fa920052070007ae305013)"
# level=info ts=2023-09-06T20:31:51.22245098Z caller=main.go:116 build_context="(go=go1.20.6, platform=linux/amd64, user=Action-Run-ID-5750091714, date=20230803-11:24:04, tags=unknown)"
# level=info ts=2023-09-06T20:31:51.223938768Z caller=main.go:164 msg="Starting web server for metrics" listen=:8080
# level=info ts=2023-09-06T20:31:51.372475343Z caller=reloader.go:376 msg="Reload triggered" cfg_in=/etc/prometheus/config/prometheus.yaml.gz cfg_out=/etc/prometheus/config_out/prometheus.env.yaml watched_dirs=/etc/prometheus/rules/prometheus-monitoring-kube-prometheus-prometheus-rulefiles-0
# level=info ts=2023-09-06T20:31:51.372646384Z caller=reloader.go:237 msg="started watching config file and directories for changes" cfg=/etc/prometheus/config/prometheus.yaml.gz out=/etc/prometheus/config_out/prometheus.env.yaml dirs=/etc/prometheus/rules/prometheus-monitoring-kube-prometheus-prometheus-rulefiles-0
# level=info ts=2023-09-12T21:15:41.446366628Z caller=reloader.go:376 msg="Reload triggered" cfg_in=/etc/prometheus/config/prometheus.yaml.gz cfg_out=/etc/prometheus/config_out/prometheus.env.yaml watched_dirs=/etc/prometheus/rules/prometheus-monitoring-kube-prometheus-prometheus-rulefiles-0
```

We can see that a reload was triggered. So we should see our two new rules in the Prometheus UI under 'Alerts'.

To test the first alert rule ('HostHighCpuLoad') we are going to run a container simulating a CPU stress. Open the browser, navigate to [Docker Hub](https://hub.docker.com/) and search for 'cpustress'. Choose the [first serch result](https://hub.docker.com/r/containerstack/cpustress), copy the `docker run` command from the documentation and translate it to a `kubectl` command:
```sh
# docker
docker run -it --name cpustress --rm containerstack/cpustress --cpu 4 --timeout 150s --metrics-brief

# kubectl
kubectl run cpustress --image=containerstack/cpustress -- --cpu 4 --timeout 150s --metrics-brief
```

Run the `kubectl` command and open the Prometheus UI > Alerts. You can see that the state of the 'HostHighCpuLoad' rule is switching from 'inactive' to 'pending' and after 2 minutes it switches to 'firing'. After the cpustress pod has finished (150s) the CPU usage is decreasing and the state of the rule is going back to 'inactive'.

Delete the pod when you're done:
```sh
kubectl delete pod cpustress
```


### Access Alertmanager UI

```bash
kubectl get service -n monitoring
# monitoring-kube-prometheus-alertmanager is the Alertmanager service running on port 9093

kubectl port-forward svc/monitoring-kube-prometheus-alertmanager -n monitoring 9093:9093 &
# Port forwarding Alertmanager service

http://127.0.0.1:9093
# Accessing Alertmanager UI
````

---

#### Steps to configure Alertmanager with Email Receiver
When you inspect the file [alertmanager.yaml](../1-install-prometheus-in-k8s/prometheus-stack/alertmanager.yaml) we created in demo project #1 by inspecting the stateful-set `alertmanager-monitoring-kube-prometheus-alertmanager`, you can find a volume called `config-volume` and see that it's holding the value of the Secret `alertmanager-monitoring-kube-prometheus-alertmanager-generated`. Here we can find the alertmanager configuration:

```sh
kubectl get secret alertmanager-monitoring-kube-prometheus-alertmanager-generated -n monitoring -o yaml | less
# apiVersion: v1
# data:
#   alertmanager.yaml.gz: H4sIAAAAAAAA/7xSy07DMBC8+ytWPSKV8BCXSHxAv4BjtDUbZyU/wnqdqlLFtyMnUFKJCk5cZ2d3xjN2Pu3RtwZAKCc/UaccKBVt4SkYSUVpGVriiaSFTSzebwyAk1TGbn+s4y1EDJRHtFTJdSsv+A+LAAHVDiQzpZLQk2i9AM/vsNnFPu3iwHvWJKeXyn1N7lvygKwtPN7lM8JRSSb0s+dqdiTUFXr/MBheLnZSfPW2BUVxpN3ayxYyTSSsx2rkgBI5uhPHPhmAnIpYusoHK6xs0RsAeitLppe5rB76uz78UfbT5X+ormqCi5KuiX+VP+dd4fMvUAqjR12aaEhtMx8PGNGRNDbFnl1zc6th9OYjAAD//xIx++qkAgAA
#mkind: Secret
# metadata:
#   creationTimestamp: "2023-09-06T20:31:41Z"
#   labels:
#     managed-by: prometheus-operator
#   name: alertmanager-monitoring-kube-prometheus-alertmanager-generated
#   namespace: monitoring
#   ownerReferences:
#   - apiVersion: monitoring.coreos.com/v1
#     blockOwnerDeletion: true
#     controller: true
#     kind: Alertmanager
#     name: monitoring-kube-prometheus-alertmanager
#     uid: 8095a848-9983-4c4e-bf88-b6eb6a169c0e
#   resourceVersion: "4458"
#   uid: 433fbe06-74c2-4162-a0ae-ac2630bcdca2
# type: Opaque

echo 'H4sIAAAAAAAA/7xSy07DMBC8+ytWPSKV8BCXSHxAv4BjtDUbZyU/wnqdqlLFtyMnUFKJCk5cZ2d3xjN2Pu3RtwZAKCc/UaccKBVt4SkYSUVpGVriiaSFTSzebwyAk1TGbn+s4y1EDJRHtFTJdSsv+A+LAAHVDiQzpZLQk2i9AM/vsNnFPu3iwHvWJKeXyn1N7lvygKwtPN7lM8JRSSb0s+dqdiTUFXr/MBheLnZSfPW2BUVxpN3ayxYyTSSsx2rkgBI5uhPHPhmAnIpYusoHK6xs0RsAeitLppe5rB76uz78UfbT5X+ormqCi5KuiX+VP+dd4fMvUAqjR12aaEhtMx8PGNGRNDbFnl1zc6th9OYjAAD//xIx++qkAgAA' | base64 -d | gunzip
# global:
#   resolve_timeout: 5m
# route:
#   receiver: "null"
#   group_by:
#   - namespace
#   routes:
#   - receiver: "null"
#     matchers:
#     - alertname =~ "InfoInhibitor|Watchdog"
#   group_wait: 30s
#   group_interval: 5m
#   repeat_interval: 12h
# inhibit_rules:
# - target_matchers:
#   - severity =~ warning|info
#   source_matchers:
#   - severity = critical
#   equal:
#   - namespace
#   - alertname
# - target_matchers:
#   - severity = info
#   source_matchers:
#   - severity = warning
#   equal:
#   - namespace
#   - alertname
# - target_matchers:
#   - severity = info
#   source_matchers:
#   - alertname = InfoInhibitor
#   equal:
#   - namespace
# receivers:
# - name: "null"
# templates:
# - /etc/alertmanager/config/*.tmpl
```

As you see this is pretty much the same content that was displayed in the Alertmanager UI. So this is were the default config comes from.

But again we don't have to adjust this configuration directly, but rather use a custom K8s compontent `AlertmanagerConfig` that is made available by the Prometheus operator.

So create a file called `alert-manager-configuration.yaml` and add the following content:
```yaml
apiVersion: monitoring.coreos.com/v1alpha1  # <-- the version may change to v1 in the near future
kind: AlertmanagerConfig
metadata:
  name: main-rules-alert-config
  namespace: monitoring
spec:
  route:
    receiver: 'email'
    repeatInterval: 30m
    routes:
    - matchers:
      - name: alertname
        value: HostHighCpuLoad
    - matchers:
      - name: alertname
        value: KubernetesPodCrashLooping
      repeatInterval: 10m
  receivers:
  - name: 'email'
    emailConfigs:
    - to: 'email@example.come'
      from: 'email@example.come'
      smarthost: 'smtp.gmail.com:587'
      authUsername: 'email@example.come'
      authIdentity: 'email@example.come'
      authPassword:
       name: gmail-auth
       key: password
```

Again to find out the structure of the `spec` attribute, check the [documentation](https://docs.openshift.com/container-platform/4.13/rest_api/monitoring_apis/alertmanagerconfig-monitoring-coreos-com-v1beta1.html) of the custom component `AlertmanagerConfig`.

The `authPassword` attribute of the 'email' receiver configuration references a Secret called `gmail-auth` with an attribute `password`. So we have to create such a Secret. For that purpose add a file called `email-secret.yaml` with the following content:

```yaml
apiVersion: v1
kind: Secret
type: Opaque
metadata: 
  name: gmail-auth
  namespace: monitoring
data:
  password: <base64-encoded-value-of-the-generated-app-password>
```

We use our gmail account as an SMTP server to send e-mails. For the alertmanager to be able to authenticate with the server we need to configure the account for 2 factor authentication and add/generate an app password which we will then use in the Secret referenced from the AlertmanagerConfig.

Login to your gmail account, click on the profile circle in the right upper corner, press "Manage your Google Account", click on "Security" in the menu on the left, scroll down to the "How you sign in to Google" section and click on the "2-Step Verification" area, turn it on and configure a second factor if it is not yet activated. To generate an App password, type "App password" in the search bar at the top of the page, select 'App passwords', click on 'Choose app' > 'Other' and type in 'K8s Prometheus Alertmanager', click on "Generate", copy the value and write it into the above Secret configuration (base64 encoded).

Apply the two configuration files to the K8s cluster:
```sh
kubectl apply -f email-secret.yaml
kubectl apply -f alert-manager-configuration.yaml
```

Check the log of the alertmanager config-reloader:
```sh
kubectl logs alertmanager-monitoring-kube-prometheus-alertmanager-0 -n monitoring -c config-reloader
# ...
# level=info ts=2023-09-14T21:46:03.943082112Z caller=reloader.go:376 msg="Reload triggered" cfg_in=/etc/alertmanager/config/alertmanager.yaml.gz cfg_out=/etc/alertmanager/config_out/alertmanager.env.yaml watched_dirs=/etc/alertmanager/config
```

Open the Alertmanager UI and navigate to the status page to see the extended config:
```yaml
global:
  resolve_timeout: 5m
  http_config:
    follow_redirects: true
    enable_http2: true
  smtp_hello: localhost
  smtp_require_tls: true
  pagerduty_url: https://events.pagerduty.com/v2/enqueue
  opsgenie_api_url: https://api.opsgenie.com/
  wechat_api_url: https://qyapi.weixin.qq.com/cgi-bin/
  victorops_api_url: https://alert.victorops.com/integrations/generic/20131114/alert/
  telegram_api_url: https://api.telegram.org
  webex_api_url: https://webexapis.com/v1/messages
route:
  receiver: "null"
  group_by:
  - namespace
  continue: false
  routes:
  - receiver: monitoring/main-rules-alert-config/email  # <-- ns/AlertmanagerConfig.metadata.name/receiver name
    matchers:
    - namespace="monitoring"  # <-- added automatically; that's why we had to add this label in the configuration file for the PrometheusRule
    continue: true
    routes:
    - match:
        alertname: HostHighCpuLoad
      continue: false
    - match:
        alertname: KubernetesPodCrashLooping
      continue: false
      repeat_interval: 10m
    repeat_interval: 30m
  - receiver: "null"
    matchers:
    - alertname=~"InfoInhibitor|Watchdog"
    continue: false
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 12h
inhibit_rules:
- source_matchers:
  - severity="critical"
  target_matchers:
  - severity=~"warning|info"
  equal:
  - namespace
  - alertname
- source_matchers:
  - severity="warning"
  target_matchers:
  - severity="info"
  equal:
  - namespace
  - alertname
- source_matchers:
  - alertname="InfoInhibitor"
  target_matchers:
  - severity="info"
  equal:
  - namespace
receivers:
- name: "null"
- name: monitoring/main-rules-alert-config/email  # <--
  email_configs:
  - send_resolved: false  # <-- if set to true, an email is sent when the problem has been resolved
    to: <gmail-address>
    from: alertmanager@prometheus.com
    hello: localhost
    smarthost: smtp.gmail.com:587
    auth_username: <gmail-address>
    auth_password: <secret>
    auth_identity: <gmail-address>
    headers:  # <--
      From: alertmanager@prometheus.com
      Subject: '{{ template "email.default.subject" . }}'
      To: <gmail-address>
    html: '{{ template "email.default.html" . }}'
    require_tls: true
templates:
- /etc/alertmanager/config/*.tmpl
```

To test the email receiver we have to create some CPU load. We do this again using the `cpustress` tool:
```sh
kubectl run cpustress --image=containerstack/cpustress -- --cpu 4 --timeout 60s --metrics-brief
# pod/cpustress created
```

After some time the average CPU load is high enough to trigger the firing of the 'HostHighCpuLoad' alert. In Prometheus UI we see, that the alert state has changed to 'FIRING'. The Alertmanager has an endpoint where you can retrieve the alerts it recently received:

```txt
http://localhost:9093/api/v2/alerts
```

This can be very useful for debugging purposes (e.g. if you don't receive an email and want to see, whether an alert has been received and whether it triggered the receiver or not). The returned JSON string contains the following object:

```json
  {
    "annotations": {
      "description": "CPU load on host is over 50%\n Value = 98.43888888892252\n Instance = 192.168.86.219:9100\n",
      "summary": "Host CPU load high."
    },
    "endsAt": "2023-09-15T19:43:55.949Z",
    "fingerprint": "f0a35342e1eb7639",
    "receivers": [
      {
        "name": "monitoring/main-rules-alert-config/email"
      }
    ],
    "startsAt": "2023-09-15T19:39:55.949Z",
    "status": {
      "inhibitedBy": [],
      "silencedBy": [],
      "state": "active"
    },
    "updatedAt": "2023-09-15T19:39:55.952Z",
    "generatorURL": "http://monitoring-kube-prometheus-prometheus.monitoring:9090/graph?g0.expr=100+-+%28avg+by+%28instance%29+%28rate%28node_cpu_seconds_total%7Bmode%3D%22idle%22%7D%5B2m%5D%29%29+%2A+100%29+%3E+50&g0.tab=1",
    "labels": {
      "alertname": "HostHighCpuLoad",
      "instance": "192.168.86.219:9100",
      "namespace": "monitoring",
      "prometheus": "monitoring/monitoring-kube-prometheus-prometheus",
      "severity": "warning"
    }
  }
```

We see that the email receiver was selected and in our e-mail inbox we find an e-mail with the subject
```txt
[FIRING:1] monitoring (HostHighCpuLoad 192.168.25.138:9100 monitoring/monitoring-kube-prometheus-prometheus warning)
```
and the content
```txt
Content:
[1] Firing
Labels
alertname = HostHighCpuLoad
instance = 192.168.86.219:9100
namespace = monitoring
prometheus = monitoring/monitoring-kube-prometheus-prometheus
severity = warning
Annotations
description = CPU load on host is over 50% Value = 98.43888888892252 Instance = 192.168.86.219:9100 
summary = Host CPU load high.
```

If the alert was received and routed to the right receiver but you still don't get an e-mail, there might be a connection or authentication problem between the alertmanager and the smtp server. You may get more information about this problem by inspecting the log of the alertmanager:
```sh
kubectl logs alertmanager-monitoring-kube-prometheus-alertmanager-0 -n monitoring -c alertmanager
```

The `cpustress` pod is restarting each time it finished its work and stopped:

```sh
kubectl get pods
# NAME                                     READY   STATUS             RESTARTS        AGE
# ...
# cpustress                                0/1     CrashLoopBackOff   6 (3m12s ago)   11m
# ...
```

After 6 restarts the second alert (`KubernetesPodCrashLooping`) is firing. The alertmanager endpoint 'http://localhost:9093/api/v2/alerts' now contains the following object:

```json
  {
    "annotations": {
      "description": "Pod cpustress is crash looping\n Value = 6",
      "summary": "Kubernetes pod crash looping."
    },
    "endsAt": "2023-09-15T19:59:25.949Z",
    "fingerprint": "7ce1e6e4c66ed194",
    "receivers": [
      {
        "name": "monitoring/main-rules-alert-config/email"
      }
    ],
    "startsAt": "2023-09-15T19:55:25.949Z",
    "status": {
      "inhibitedBy": [],
      "silencedBy": [],
      "state": "active"
    },
    "updatedAt": "2023-09-15T19:55:25.952Z",
    "generatorURL": "http://monitoring-kube-prometheus-prometheus.monitoring:9090/graph?g0.expr=kube_pod_container_status_restarts_total+%3E+5&g0.tab=1",
    "labels": {
      "alertname": "KubernetesPodCrashLooping",
      "container": "cpustress",
      "endpoint": "http",
      "instance": "192.168.2.83:8080",
      "job": "kube-state-metrics",
      "namespace": "monitoring",
      "pod": "cpustress",
      "prometheus": "monitoring/monitoring-kube-prometheus-prometheus",
      "service": "monitoring-kube-state-metrics",
      "severity": "critical",
      "uid": "95574a40-f553-4084-8337-4d79a4cb5df3"
    }
  }
```

And we received a second e-mail with the subject
```txt
[FIRING:1] monitoring (KubernetesPodCrashLooping cpustress http 192.168.2.83:8080 kube-state-metrics cpustress monitoring/monitoring-kube-prometheus-prometheus monitoring-kube-state-metrics critical 95574a40-f553-4084-8337-4d79a4cb5df3)
```

and the content
```txt
[1] Firing
Labels
alertname = KubernetesPodCrashLooping
container = cpustress
endpoint = http
instance = 192.168.2.83:8080
job = kube-state-metrics
namespace = monitoring
pod = cpustress
prometheus = monitoring/monitoring-kube-prometheus-prometheus
service = monitoring-kube-state-metrics
severity = critical
uid = 95574a40-f553-4084-8337-4d79a4cb5df3
Annotations
description = Pod cpustress is crash looping Value = 6
summary = Kubernetes pod crash looping.
```

Now we can delete the cpustress pod to stop the alerts:
```sh
kubectl delete pod cpustress
```


---

## Allowing Applications to Send Emails via Email Providers

### 1. Two-Step Authentication (Recommended)

Enable 2-Step Authentication on your email account and generate an **App Password**.

### 2. Less Secure Apps (Alternative)

Enable the "Allow less secure apps" setting (deprecated in many Gmail accounts).
Link: [https://myaccount.google.com/lesssecureapps](https://myaccount.google.com/lesssecureapps)

---



## Key Fixes Needed

### 1. Enable 2-Step Verification & Use App Password

```bash
kubectl create secret generic gmail-auth -n monitoring \
  --from-literal=pass=your_app_password_here
```

---

### 2. Verify SMTP Configuration

Ensure the following:

```yaml
emailConfigs:
  - smarthost: smtp.gmail.com:587
    auth_identity: awais.akram11199@gmail.com
    auth_username: awais.akram11199@gmail.com
```

---

### 3. Check Secret Deployment

```bash
kubectl get secret gmail-auth -n monitoring -o yaml
```

---

### 4. Troubleshooting Steps

Test SMTP credentials independently using `swaks`:

```bash
swaks --to awais.akram11199@gmail.com --from awais.akram11199@gmail.com \
  --server smtp.gmail.com:587 --auth LOGIN \
  --auth-user awais.akram11199@gmail.com --auth-password YOUR_APP_PASSWORD
```

> Ensure no IP blocking exists (Google may block unfamiliar locations).

---

## Test Email Notifications

When a high CPU alert fires, Alertmanager receives it and routes it to the configured email based on its `alertname` and `namespace` labels. The resulting email notification confirms the setup. Similarly, a pod crash-looping alert also triggers and gets routed to email, demonstrating the label-based routing in action for proactive cluster monitoring.

---

## License

MIT License
