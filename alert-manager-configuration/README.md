# AlertManager Configuration

When an alert rule in Prometheus goes into a "firing" state, it means Prometheus has detected that the defined condition (like high CPU load) has been met. Prometheus then sends this alert to Alertmanager. 

![Alert Flow Diagram](#) <!-- Replace '#' with your actual image path -->

However, just because an alert is firing in Prometheus doesn't automatically mean you'll get a notification. Alertmanager is the component responsible for actually routing and sending out notifications (like emails). In the scenario described, Prometheus fired an alert, but since Alertmanager wasn't configured, the alert was essentially ignored, and no notification was sent. We'll configure Alertmanager to handle these firing alerts and send out email notifications.

---

## Alertmanager Configuration File

Alertmanager is deployed by the Prometheus operator as an independent/dedicated application like we deployed Prometheus. So Prometheus is one application, Alertmanager is another application. And that means Alertmanager has its own configuration configured through a configuration file.

Alertmanager is a separate application from Prometheus, configured via its own file. Its basic UI allows viewing the configuration and filtering alerts. The configuration has three main sections:
- `global`: for overarching settings
- `routes`: to direct alerts to specific receivers based on matching labels (like `alertname`)
- `receivers`: which define the notification channels (e.g., email, Slack)

Currently, a "null" receiver is configured, causing all incoming alerts from Prometheus to be ignored. The `routes` section uses `match` attributes to specify which alerts go to which receivers. Additionally, there are grouping configurations to bundle alerts for more efficient notifications. The goal is to modify this configuration to add an email receiver and create routes for specific alerts like **"HostHighCPULoad"** and **"KubernetesPodCrashLooping"** to be sent via email.

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

## Configure Email Notifications

This section details creating an Alertmanager configuration in Kubernetes using the `AlertmanagerConfig` custom resource (`apiVersion: monitoring.coreos.com/v1alpha1`, `kind: AlertmanagerConfig`). It defines an "email" receiver with sender, recipient, and Gmail SMTP server details. For secure password management, it references a Kubernetes secret named `gmail-auth` in the `monitoring` namespace.

### `alertmanager-configurations.yaml`

```yaml
apiVersion: monitoring.coreos.com/v1alpha1  # API version for the AlertmanagerConfig CRD
kind: AlertmanagerConfig                     # Kind of the Kubernetes resource
metadata:
  name: main-rules-alert-config             # Name of the AlertmanagerConfig object
  namespace: monitoring                     # Namespace where the config applies (usually where Alertmanager is running)
spec:
  route:                                    # Top-level routing configuration
    receiver: 'email'                       # Default receiver if no child route matches
    repeatInterval: 30m                     # Time to wait before sending a repeated notification
    routes:                                 # Child routes for specific alerts
      - matchers:                           # Match alerts with specific labels
          - name: alertname
            value: HostHighCPULoad          # Matches alerts with alertname: HostHighCPULoad
      - matchers:
          - name: alertname
            value: KubernetesPodCrashLooping  # Matches alerts with alertname: KubernetesPodCrashLooping
        receiver: 'email'                   # Receiver for this route
        repeatInterval: 15m                 # Repeat interval for this specific alert

  receivers:                                # List of configured notification receivers
    - name: 'email'                         # Receiver name to be used in routes
      emailConfigs:
        - to: 'awais.akram@gmail.com'       # Email recipient
          from: 'awais.akram@gmail.com'     # Sender email
          smarthost: 'smtp.gmail.com:587'   # SMTP server for Gmail
          authUsername: 'awais.akram@gmail.com'  # SMTP auth username
          authIdentity: 'awais.akram@gmail.com'  # SMTP auth identity (often same as username)
          authPassword:                     # Reference to a Kubernetes secret for the SMTP password
            name: gmail-auth                # Name of the secret object
            key: pass                       # Key within the secret that holds the password

```

---

### `email-secret.yaml`

> Creating a secret for your Gmail password. **Do not check it into your repo.**

```yaml
apiVersion: v1                             # Kubernetes API version for standard resources
kind: Secret                               # Resource type is a Secret
metadata:
  name: gmail-auth                         # Name of the secret object
  namespace: monitoring                    # Namespace where the secret will be created
type: Opaque                               # Indicates this is a generic secret (base64-encoded data)
data:
  pass: your-password-base64-encoded-value # Replace with your actual base64-encoded Gmail app password

```

```bash
# Encode your Gmail password
echo -n 'your-gmail-password' | base64

# Decode (for testing)
echo -n 'base64-encoded-password-value' | base64 -d
```

---

## Allowing Applications to Send Emails via Email Providers

### 1. Two-Step Authentication (Recommended)

Enable 2-Step Authentication on your email account and generate an **App Password**.

### 2. Less Secure Apps (Alternative)

Enable the "Allow less secure apps" setting (deprecated in many Gmail accounts).
Link: [https://myaccount.google.com/lesssecureapps](https://myaccount.google.com/lesssecureapps)

---

## Apply the Configurations

```bash
kubectl apply -f email-secret.yaml
kubectl apply -f alertmanager-configurations.yaml

kubectl get alertmanagerconfig -n monitoring
# Confirm the configuration was applied
```

Alertmanager should automatically reload this configuration.

### View Logs

```bash
kubectl get pod -n monitoring

# The Alertmanager Pod has two containers:
# 1. config-reloader
# 2. alertmanager

kubectl logs alertmanager-monitoring-kube-prometheus-alertmanager-0 -n monitoring -c config-reloader
# Logs for config-reloader

kubectl logs alertmanager-monitoring-kube-prometheus-alertmanager-0 -n monitoring -c alertmanager
# Logs for alertmanager
```

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
