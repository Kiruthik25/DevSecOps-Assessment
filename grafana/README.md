# Kubernetes Monitoring with Prometheus and Grafana

This guide explains how to install `kube-prometheus-stack`, open
Grafana, import dashboards, configure a custom Kubernetes pod-health
dashboard, and enable email notifications.

## Contents

-   [Prerequisites](#prerequisites)
-   [Install kube-prometheus-stack](#1-install-kube-prometheus-stack)
-   [Verify the installation](#2-verify-the-installation)
-   [Open Grafana](#3-open-grafana)
-   [Import community dashboards](#4-import-community-dashboards)
-   [Apply the custom pod-health
    dashboard](#5-apply-the-custom-pod-health-dashboard)
-   [Alert rules](#6-alert-rules)
-   [Configure email notifications](#7-configure-email-notifications)
-   [Troubleshooting](#8-troubleshooting)

## Prerequisites

Before starting, make sure you have:

-   A running Kubernetes cluster and a working `kubectl` context.
-   Helm 3 installed.
-   Permission to create resources in the cluster.
-   The repository's `k8s-pod-healht.yaml` file and `alerting/` folder
    available locally.

**Why this step matters:** Checking these prerequisites first helps
avoid installation failures caused by a missing cluster context, Helm,
or configuration files.

Check your cluster and tools:

``` bash
kubectl cluster-info
kubectl get nodes
helm version
```

## 1. Install kube-prometheus-stack

The `kube-prometheus-stack` Helm chart installs Prometheus, Grafana,
Alertmanager, node-exporter, kube-state-metrics, and related monitoring
resources.

Add the Prometheus Community chart repository and update it:

``` bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

Install the stack in the `monitoring` namespace:

``` bash
helm upgrade --install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace
```

> **Important:** The Helm release name used in this guide is
> `monitoring`. If you choose another release name, update the
> release-specific commands accordingly.

### Prometheus data source is configured automatically

When Grafana is installed through `kube-prometheus-stack`, the chart
provisions a Prometheus data source automatically. You normally **do not
need to add the Prometheus data source manually**. Imported dashboards
can use this provisioned data source.

## 2. Verify the installation

Check the Helm release and the pods:

``` bash
helm list -n monitoring
kubectl get pods -n monitoring
```

Wait for the monitoring workloads to become ready. You can also inspect
the Grafana deployment:

``` bash
kubectl get deployments -n monitoring
kubectl rollout status deployment/monitoring-grafana -n monitoring
```

The Grafana deployment name may differ if you use a different Helm
release name.

## 3. Open Grafana

Forward the Grafana service to your local machine:

``` bash
kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80
```

Then open <http://localhost:3000>.

The default username is usually `admin`. Retrieve the generated password
with:

``` bash
kubectl get secret monitoring-grafana -n monitoring \
  -o jsonpath="{.data.admin-password}" | base64 --decode
echo
```

If the secret or service name differs, list the resources in the
`monitoring` namespace and use the names from your installation.

## 4. Import community dashboards

Grafana dashboards can be imported using their dashboard IDs.

1.  Open Grafana.
2.  Navigate to **Dashboards → New → Import** (the menu wording can vary
    by Grafana version).
3.  Enter one of the IDs below and select **Load**.
4.  When prompted, select the provisioned **Prometheus** data source.
5.  Click **Import**.

  ------------------------------------------------------------------------
  Dashboard                     Grafana dashboard ID Purpose
  --------------------- ---------------------------- ---------------------
  Node Exporter Full                          `1860` Host/node CPU,
                                                     memory, filesystem,
                                                     disk, and network
                                                     metrics

  Kubernetes Pod                             `15760` Pod and container
  Overview                                           resource usage
  ------------------------------------------------------------------------

Dashboard links:

-   [Node Exporter Full --- ID
    1860](https://grafana.com/grafana/dashboards/1860-node-exporter-full/)
-   [Kubernetes Pod Overview --- ID
    15760](https://grafana.com/grafana/dashboards/15760-kubernetes-pod-overview/)

If a dashboard shows **No data**, confirm that Prometheus targets are
healthy and that the dashboard's variables and data source match the
metrics available in your cluster.

## 5. Apply the custom pod-health dashboard

Use the repository file `k8s-pod-healht.yaml` to create the custom
Kubernetes pod-health dashboard configuration.

Apply it:

``` bash
kubectl apply -f k8s-pod-healht.yaml
```

Check that the resource was created:

``` bash
kubectl get configmaps -n monitoring
```

The file must contain a valid Kubernetes manifest and use the namespace
and labels expected by your Grafana dashboard provisioning setup. If the
manifest specifies a namespace, ensure that namespace exists and is
intentional.

**Why this step matters:** A custom pod-health dashboard lets you focus
on the pod-level signals that matter to your cluster instead of relying
only on general-purpose dashboards.

> **Note:** If `k8s-pod-healht.yaml` is a Grafana dashboard JSON file
> rather than a Kubernetes manifest, `kubectl apply` is not the correct
> method. Import the dashboard through **Dashboards → New → Import**, or
> use the Grafana dashboard provisioning configuration that matches your
> repository.

## 6. Alert rules

The repository's `alerting/` folder contains alert-rule configuration
for monitoring conditions that may need attention.

Review the rule files before applying them. Confirm that the expressions
use metrics available in your cluster, and check the target namespace
and rule-group configuration. Apply a rule file only using the method
expected by its resource type---for example, Kubernetes `PrometheusRule`
manifests can be applied with `kubectl apply -f <file.yaml>`.

**Why this step matters:** Alert rules turn observed metrics into
actionable signals when a defined condition is met, rather than
requiring someone to watch dashboards continuously.

## 7. Configure email notifications

Grafana OSS requires SMTP configuration before it can send email
notifications. This section uses Gmail as an example; another SMTP
provider can be used by changing the server settings.

### 7.1 Find your Helm release

``` bash
helm list -n monitoring
```

The commands below assume the release name is `monitoring`.

### 7.2 Create a Gmail app password

If you use Gmail:

1.  Enable two-step verification on your Google account.
2.  Generate an **App Password** in your Google Account security
    settings.
3.  Use that app password for SMTP---not your normal Gmail password.

Keep the app password private. Do not commit it to Git or place it
directly in a values file.

### 7.3 Store the SMTP password in a Kubernetes Secret

Replace the example value with your Gmail app password:

``` bash
kubectl create secret generic grafana-smtp-secret \
  --namespace monitoring \
  --from-literal=GF_SMTP_PASSWORD='YOUR_GMAIL_APP_PASSWORD'
```

If the Secret already exists, update it using your preferred secure
secret-management workflow rather than repeatedly running the create
command.

### 7.4 Create a Grafana SMTP values file

Create `grafana-smtp-values.yaml`:

``` yaml
grafana:
  envFromSecret: grafana-smtp-secret

  grafana.ini:
    smtp:
      enabled: true
      host: smtp.gmail.com:587
      user: your-email@gmail.com
      password: "$__env{GF_SMTP_PASSWORD}"
      from_address: your-email@gmail.com
      from_name: Grafana
```

Replace both occurrences of `your-email@gmail.com` with your Gmail
address. Keep the SMTP password in the Kubernetes Secret.

### 7.5 Apply the configuration

``` bash
helm upgrade monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --reuse-values \
  -f grafana-smtp-values.yaml
```

If your release has a different name, replace `monitoring` accordingly.

Check the Grafana rollout:

``` bash
kubectl rollout status deployment/monitoring-grafana -n monitoring
```

If you are unsure of the deployment name:

``` bash
kubectl get deployments -n monitoring | grep grafana
```

### 7.6 Create an email contact point in Grafana

1.  In Grafana, open **Alerting → Contact points**. The menu may appear
    under **Notification configuration** in some versions.
2.  Select **New contact point**.
3.  Set the name to `DevOpsEmailAlerts`.
4.  Select **Email** as the integration.
5.  Enter the recipient email address.
6.  Use **Test** to send a test notification.
7.  Confirm that the email arrives, then save the contact point.

A contact point defines where notifications are delivered. Ensure that
your alert rules or notification policies are configured to route
matching alerts to this contact point.

## 8. Troubleshooting

### Grafana pod is not ready

``` bash
kubectl get pods -n monitoring
kubectl describe pod -n monitoring <grafana-pod-name>
kubectl logs -n monitoring <grafana-pod-name> --all-containers=true
kubectl get events -n monitoring --sort-by=.lastTimestamp
```

Look for image-pull errors, storage/PVC issues, resource constraints, or
permission errors.

### Dashboard displays no data

-   Confirm Prometheus and its targets are healthy.
-   Confirm the dashboard uses the provisioned Prometheus data source.
-   Check whether the required metrics are exposed and scraped.
-   Adjust dashboard variables such as cluster, namespace, node, or pod
    to match your environment.

### Email test fails

-   Confirm the SMTP Secret exists in the `monitoring` namespace.
-   Verify the Gmail address and app password.
-   Confirm outbound access to `smtp.gmail.com:587`.
-   Inspect Grafana logs for SMTP authentication or connection errors.

------------------------------------------------------------------------

## Useful commands

``` bash
# Check monitoring resources
kubectl get all -n monitoring

# Check Helm release values
helm get values monitoring -n monitoring

# View Grafana logs
kubectl logs -n monitoring deployment/monitoring-grafana

# Check PrometheusRule resources
kubectl get prometheusrules -A
```

**Security reminder:** Keep SMTP credentials out of source control and
rotate the app password if it is exposed.
