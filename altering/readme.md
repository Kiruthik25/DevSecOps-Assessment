# Kubernetes Alerting with Prometheus and Alertmanager

This guide continues from the monitoring setup: after configuring email
notifications in Grafana, the next step is to create **Prometheus alert
rules** and configure **Alertmanager routing** so selected alerts are
delivered to email.

## Why alert rules and routing?

-   **Alert rules** define the conditions that should trigger an alert,
    such as a pod staying unready or a container being OOMKilled.
-   **Alert routing** decides where each alert should go, so the right
    team or receiver gets the notification instead of every alert being
    sent to everyone.

## Overview

This setup creates two Prometheus alerts:

  -----------------------------------------------------------------------
  Alert                               Condition
  ----------------------------------- -----------------------------------
  `KubernetesPodNotReady`             A pod remains unready for 5
                                      minutes.

  `KubernetesContainerOOMKilled`      A container is OOMKilled and its
                                      restart counter increases within
                                      the configured 10-minute window.
  -----------------------------------------------------------------------

The rules are defined in `kubernetes-pod-health-alerts.yaml`.
Alertmanager routing is configured separately in
`alertmanager-routing-values.yaml`.

> **Important:** These are Kubernetes `PrometheusRule` alerts managed by
> Prometheus. They are not Grafana-managed alert rules. Grafana's own
> SMTP configuration does not configure Alertmanager's SMTP settings.

## Prerequisites

-   `kube-prometheus-stack` is installed in the `monitoring` namespace.
-   `kubectl` and Helm are configured for the correct cluster.
-   The `kubernetes-pod-health-alerts.yaml` and
    `alertmanager-routing-values.yaml` files are available locally.
-   Gmail two-step verification is enabled and a Gmail App Password has
    been generated, if using Gmail.

Check the Helm release name:

``` bash
helm list -n monitoring
```

The commands below assume the release is named `monitoring`. Replace it
if your release uses a different name.

## Step 1 --- Apply the Prometheus alert rules

Apply the `PrometheusRule` manifest:

``` bash
kubectl apply -f kubernetes-pod-health-alerts.yaml
```

Verify that the rule exists:

``` bash
kubectl get prometheusrules -n monitoring
kubectl describe prometheusrule kubernetes-pod-health-alerts -n monitoring
```

The resource name in the `describe` command must match `metadata.name`
in your YAML.

### Check that Prometheus recognizes the alerts

Forward the Prometheus service:

``` bash
kubectl port-forward -n monitoring \
  svc/monitoring-kube-prometheus-prometheus 9090:9090
```

Open <http://localhost:9090/alerts> and confirm that both alert rules
appear. The service name may differ in your installation; find it with:

``` bash
kubectl get svc -n monitoring | grep prometheus
```

If the rules do not appear, verify the `PrometheusRule` labels and the
`ruleSelector` configuration of your Prometheus instance. In a default
`kube-prometheus-stack` installation, the rule's `release` label
commonly needs to match the Helm release name. For this guide, that is
`monitoring`.

## Step 2 --- Create the Alertmanager SMTP password Secret

This example uses Gmail. Generate an App Password in your Google Account
security settings; do not use your normal Gmail password.

``` bash
kubectl create secret generic alertmanager-smtp-secret \
  --namespace monitoring \
  --from-literal=smtp-auth-password='YOUR_GMAIL_APP_PASSWORD'
```

Keep this password out of Git and avoid printing it in terminal logs. If
the Secret already exists, update it using your team's approved
secret-management workflow.

## Step 3 --- Configure Alertmanager routing

Create or review `alertmanager-routing-values.yaml`. The example below
mounts the Secret into Alertmanager and routes only the two named alerts
to email. Unmatched alerts go to a `null` receiver, so they are not
emailed by this configuration.

Replace `your-email@gmail.com` with the Gmail account used to send mail
and `YOUR_RECEIVING_EMAIL@gmail.com` with the destination address.

``` yaml
alertmanager:
  alertmanagerSpec:
    secrets:
      - alertmanager-smtp-secret

  config:
    global:
      smtp_smarthost: "smtp.gmail.com:587"
      smtp_from: "your-email@gmail.com"
      smtp_auth_username: "your-email@gmail.com"
      smtp_auth_password_file: "/etc/alertmanager/secrets/alertmanager-smtp-secret/smtp-auth-password"
      smtp_require_tls: true

    route:
      receiver: "null"
      group_by:
        - alertname
        - namespace
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 4h
      routes:
        - receiver: "DevOpsEmailAlerts"
          matchers:
            - alertname="KubernetesPodNotReady"
        - receiver: "DevOpsEmailAlerts"
          matchers:
            - alertname="KubernetesContainerOOMKilled"

    receivers:
      - name: "null"
      - name: "DevOpsEmailAlerts"
        email_configs:
          - to: "YOUR_RECEIVING_EMAIL@gmail.com"
            send_resolved: true
```

### Understand the routing configuration

-   `global`: defines Alertmanager's SMTP connection settings.
-   `route`: defines the default receiver and the rules for matching
    alerts.
-   `DevOpsEmailAlerts`: sends the two selected alert types to the
    configured email address.
-   `null`: discards alerts that do not match the two explicit routes in
    this example.
-   `send_resolved: true`: sends a notification when an alert returns to
    its resolved state.

> **Warning:** This example intentionally replaces the Alertmanager
> configuration supplied through `alertmanager.config`. If you already
> use custom routes, receivers, inhibition rules, or other alert
> destinations, merge them into this configuration before applying it.
> Do not overwrite a working production configuration without reviewing
> it.

## Step 4 --- Apply the Alertmanager configuration

Apply the values file with Helm:

``` bash
helm upgrade monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --reuse-values \
  -f alertmanager-routing-values.yaml
```

Wait for the Alertmanager rollout to complete. Find the StatefulSet name
first if needed:

``` bash
kubectl get statefulsets -n monitoring | grep alertmanager
```

Then check its rollout status, using the actual StatefulSet name:

``` bash
kubectl rollout status statefulset/alertmanager-monitoring-kube-prometheus-alertmanager \
  -n monitoring
```

The StatefulSet name varies by release and chart version. If the example
name does not exist, use the name returned by
`kubectl get statefulsets -n monitoring`.

## Step 5 --- Verify Alertmanager and routing

Check the Alertmanager pods:

``` bash
kubectl get pods -n monitoring | grep alertmanager
```

Forward the Alertmanager service:

``` bash
kubectl port-forward -n monitoring \
  svc/monitoring-kube-prometheus-alertmanager 9093:9093
```

Open <http://localhost:9093> and inspect the Alerts and Status pages.
Service names can vary; find the correct one with:

``` bash
kubectl get svc -n monitoring | grep alertmanager
```

Check Alertmanager logs if needed:

``` bash
kubectl logs -n monitoring <alertmanager-pod-name> --all-containers=true
```

### Expected routing

  Alert name                       Destination
  -------------------------------- ---------------------------------
  `KubernetesPodNotReady`          Configured recipient email
  `KubernetesContainerOOMKilled`   Configured recipient email
  All other alerts                 `null` receiver in this example

The alert names in the routes must match the `alertname` values emitted
by the Prometheus rules exactly. If you want to receive other
`kube-prometheus-stack` alerts, add routes for them or change the
default receiver.

## Troubleshooting

### The rules do not appear in Prometheus

-   Confirm the `PrometheusRule` was applied in the expected namespace.
-   Check the rule's labels, especially the `release` label.
-   Check the Prometheus `ruleSelector` and namespace selection.
-   Review the Prometheus UI for rule evaluation errors.

### Alerts appear in Prometheus but no email arrives

-   Confirm the alerts are firing, not just pending or inactive.
-   Confirm the Alertmanager routes match the exact alert names.
-   Verify the SMTP username, Gmail App Password, and destination
    address.
-   Confirm the Secret is mounted and the password-file path matches the
    YAML.
-   Check Alertmanager logs for SMTP authentication, TLS, or
    connectivity errors.
-   Confirm that network policies or firewall rules allow outbound SMTP
    traffic to `smtp.gmail.com:587`.

### Confirm the Secret exists (without displaying its value)

``` bash
kubectl get secret alertmanager-smtp-secret -n monitoring
```

## Summary

1.  Apply `kubernetes-pod-health-alerts.yaml` to define the Prometheus
    alert rules.
2.  Create the SMTP Secret without committing credentials to source
    control.
3.  Configure `alertmanager-routing-values.yaml` to send the selected
    alerts to email.
4.  Apply the values with Helm and verify the rules, routes, and email
    delivery.

**Key takeaway:** Alert rules detect a problem; routing ensures the
notification reaches the correct destination.
