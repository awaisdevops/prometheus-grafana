# Create Own Alert Rules

This project implements custom alert rules within a Kubernetes cluster to provide proactive notifications for critical events. The following alert rules are defined:

![Alert Rules Diagram](image.png)



1. **High CPU Usage**: An alert will be triggered if the CPU utilization on any Kubernetes node surpasses 50%. This helps identify potential performance bottlenecks.  
2. **Pod Startup Failure**: The system will alert if a pod is unable to start and enters a 'crash loop' state, indicating potential issues with the pod's configuration or dependencies.

---

## Create First Rule

We'll create and implement a custom Kubernetes alert rule for detecting high CPU load on nodes, defined within the `alert_rules.yaml` file. The `host_high_cpu_load` alert triggers a warning if a node's average CPU usage, calculated over a two-minute window, exceeds 50% for a sustained period. The alert message includes the actual CPU usage percentage and the name of the affected instance, providing context for troubleshooting. Additionally, the alert is labeled with `namespace: monitoring`, which can be used for targeted management within Alertmanager.

---

## Create Second Rule

We'll add a second rule **"Kubernetes pod crash looping"** alert within a `PrometheusRule` resource. This alert triggers immediately (`for: 0m`) if a pod's container restart count (`kube_pod_container_status_restarts_total`) exceeds five. It's labeled as `critical` and includes annotations detailing the crashing pod's name (extracted from the `pod` label) and the number of restarts. This demonstrates how to define an alert for a specific Kubernetes condition within the Prometheus Operator's custom resource.

---

## Configuring Prometheus Alert Rules with the Prometheus Operator

The Prometheus Operator simplifies alert rule management in Kubernetes by enabling the definition of rules as custom Kubernetes resources (`PrometheusRule`). Instead of manual configuration file edits, we'll create YAML files defining these resources, which include specifications for alert groups and individual alerts (name, expression, duration, labels, annotations). The operator automatically detects and applies these rules to Prometheus, handling configuration updates without manual intervention or restarts. This Kubernetes-native method streamlines alert configuration and prevents manual modifications to Prometheus's rule files.

---

## `alert-rules.yaml` Configuration

```yaml
apiVersion: monitoring.coreos.com/v1  # API version for PrometheusRule custom resource
kind: PrometheusRule                  # Defines a Prometheus alert rule set
metadata:
  name: main-rules                    # Name of the PrometheusRule resource
  namespace: monitoring               # Namespace where the rule is deployed
  labels:
    app: kube-prometheus-stack        # Label to associate with Prometheus stack
    release: monitoring               # Label to identify release grouping

spec:
  groups:
  - name: main.rules                  # Group name for organizing rules
    rules:
    - alert: HostHighCPULoad          # First alert rule: high CPU usage on nodes
      expr: 100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[2m])) * 100) > 50
      # Expression: calculates average CPU usage per node over 2 minutes. Triggers if usage > 50%
      for: 2m                          # Alert fires only if condition is sustained for 2 minutes
      labels:
        severity: warning             # Label used for alert severity
        namespace: monitoring         # Label indicating monitoring namespace
      annotations:
        summary: "CPU load on the host is over 50%\n Value = {{ $value }}"  # Short summary
        description: "The CPU load on node {{ $labels.instance }} has been above 50% for more than 2 minutes."  # Detailed description
        runbook_url: "http://your.runbook.url"                             # URL to runbook for resolving alert
        grafana_dashboard_url: "http://your.grafana.dashboard.url"        # URL to relevant Grafana dashboard

    - alert: KubernetesPodCrashLooping  # Second alert rule: crash looping pods
      expr: kube_pod_container_status_restarts_total > 5
      # Expression: triggers if container restart count exceeds 5
      for: 0m                            # Fires immediately once condition is met
      labels:
        severity: critical              # More severe label for crash loops
        namespace: monitoring
      annotations:
        description: "Pod {{ $labels.pod }} is crash looping\n Value = {{ $value }}"  # Detailed info with pod name
        summary: "Kubernetes pod is crash looping"                                     # Short summary
````

---

## Applying and Verifying Custom Alert Rules

After defining the `PrometheusRule`, we'll apply them to the Kubernetes cluster using `kubectl apply`. The creation of the `PrometheusRule` resource can be verified with `kubectl get prometheusrule -n monitoring`. The Prometheus Operator then automatically detects this new resource and triggers a configuration reload for the Prometheus pod. This reload can be confirmed by examining the logs of the `config-reloader` container within the Prometheus pod. Successful reloading ensures that Prometheus picks up the newly defined alert rules, which will then be visible and active in the Prometheus UI under the specified group name (e.g., "main rules").

```bash
# First install Prometheus CRDs
kubectl apply -f https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/main/example/prometheus-operator-crd/monitoring.coreos.com_prometheusrules.yaml

# Verify the CRDs
kubectl get crds | grep prometheusrules

# Output:
# prometheusrules.monitoring.coreos.com

# Apply your custom alert rules
kubectl apply -f alert-rules.yaml

# Verify PrometheusRule is created
kubectl get PrometheusRule -n monitoring
# Output:
# printing prometheus monitoring rules (main-rules)

# Also check your added rule on Prometheus UI in alerts.
```

Check Prometheus pod logs to ensure rule loading:

```bash
# List pods in monitoring namespace
kubectl get pod -n monitoring

# Output (example):
# prometheus-monitoring-kube-prometheus-prometheus-0

# Get logs from Prometheus pod
kubectl logs prometheus-monitoring-kube-prometheus-prometheus-0 -n monitoring

# You may be prompted to specify container (prometheus or config-reloader)

# Logs for sidecar container (config-reloader)
kubectl logs prometheus-monitoring-kube-prometheus-prometheus-0 -n monitoring -c config-reloader

# Logs for main Prometheus container
kubectl logs prometheus-monitoring-kube-prometheus-prometheus-0 -n monitoring -c prometheus
```

---

## License

MIT License
