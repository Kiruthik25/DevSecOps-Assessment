# Centralized Kubernetes Logging with EFK Stack on Amazon EC2

## 1. Overview

This guide describes how to implement centralized log aggregation for a Kubernetes cluster running on an Amazon EC2 instance.

The solution uses the **EFK stack — Elasticsearch, Fluent Bit, and Kibana** — to collect, store, search, and visualize application logs from Kubernetes workloads.

Elasticsearch and Kibana are managed using the Elastic Cloud on Kubernetes (ECK) operator, while Fluent Bit runs as a DaemonSet to collect container logs from Kubernetes nodes.

### Architecture

Application Pods → Fluent Bit → Elasticsearch → Kibana

### Components and Responsibilities

| Component | Responsibility |
|---|---|
| ECK Operator | Manages Elasticsearch and Kibana deployments |
| Fluent Bit | Collects container logs and forwards them to Elasticsearch |
| Elasticsearch | Stores, indexes, and enables searching of logs |
| Kibana | Provides log exploration, filtering, and dashboard visualization |

## 2. Prerequisites

Before installing the logging stack, verify that the following requirements are satisfied:

- A running Kubernetes cluster accessible through `kubectl`.
- Helm installed and configured.
- Sufficient CPU, memory, and disk capacity on the EC2 host.
- A functional Kubernetes storage configuration if persistent volumes are required.
- Network connectivity between Fluent Bit and Elasticsearch.

Check the cluster and available resources:

```bash
kubectl get nodes
kubectl get storageclass
kubectl get pods -A | grep -E 'Pending|CrashLoopBackOff|ImagePullBackOff'

free -h
df -h
```

If the Metrics Server is available, inspect node resource utilization:

```bash
kubectl top nodes
```

**Resource consideration:** Elasticsearch can consume significant memory and storage. Confirm that the EC2 instance has sufficient capacity before proceeding.

## 3. Install the ECK Operator

The ECK operator manages the lifecycle of Elasticsearch and Kibana resources in Kubernetes.

Add the Elastic Helm repository and install the operator:

```bash
helm repo add elastic https://helm.elastic.co
helm repo update

helm upgrade --install elastic-operator elastic/eck-operator \
  --namespace elastic-system \
  --create-namespace
```

Verify the installation:

```bash
kubectl get pods -n elastic-system
kubectl get crd | grep elastic
```

Proceed once the operator is running and the required custom resource definitions are available.

## 4. Deploy Elasticsearch

Elasticsearch acts as the centralized storage and indexing backend for Kubernetes application logs.

Create the Elasticsearch manifest file:

`elasticsearch.yaml`

Apply the manifest:

```bash
kubectl create namespace logging

kubectl apply -f elasticsearch.yaml
```

Monitor the deployment:

```bash
kubectl get elasticsearch -n logging -w
kubectl get pods -n logging -w
kubectl get pvc -n logging
```

Wait for Elasticsearch to become ready before proceeding to Kibana and Fluent Bit configuration.

ECK configures HTTPS by default and creates an `elastic` user whose password is stored in a Kubernetes Secret.

## 5. Deploy Kibana

Kibana provides the interface for exploring indexed logs, filtering events, and building dashboards.

Create the Kibana manifest file:

`kibana.yaml`

Apply the manifest:

```bash
kubectl apply -f kibana.yaml
```

Verify the deployment:

```bash
kubectl get kibana -n logging
kubectl get pods -n logging
```

Ensure that the Kibana and Elasticsearch versions are compatible and that both components are ready.

## 6. Configure Fluent Bit

Fluent Bit collects container logs from Kubernetes nodes, enriches them with Kubernetes metadata, and forwards them to Elasticsearch.

### 6.1 Configure Elasticsearch Authentication

Retrieve the Elasticsearch password and store it in a Kubernetes Secret for Fluent Bit:

```bash
PASSWORD=$(kubectl get secret efk-es-elastic-user \
  -n logging \
  -o go-template='{{.data.elastic | base64decode}}')

kubectl create secret generic fluent-bit-es-auth \
  -n logging \
  --from-literal=ES_PASSWORD="$PASSWORD"
```

If the Secret already exists, update it using an apply-based workflow rather than creating a duplicate.

### 6.2 Create the Fluent Bit Configuration

Create the Helm values file:

`fluent-bit-values.yaml`

This file contains the Fluent Bit input, filter, output, and authentication configuration. The configuration should:

- Read container logs from `/var/log/containers/`.
- Enrich records with Kubernetes metadata.
- Forward logs to the Elasticsearch service.
- Use the Elasticsearch credentials stored in the Secret.
- Configure buffering and retry behavior.

For this lab environment, TLS certificate verification may be disabled temporarily. For production environments, configure the Elasticsearch CA certificate and enable certificate verification.

### 6.3 Install Fluent Bit

Add the official Fluent Bit Helm repository:

```bash
helm repo add fluent https://fluent.github.io/helm-charts
helm repo update
```

Install Fluent Bit using the configuration file:

```bash
helm upgrade --install fluent-bit fluent/fluent-bit \
  --namespace logging \
  -f fluent-bit-values.yaml
```

Verify the collector:

```bash
kubectl get daemonset,pods -n logging
kubectl logs -n logging daemonset/fluent-bit --since=10m
```

Investigate authentication failures, TLS errors, indexing failures, or repeated retries before proceeding.

## 7. Access Kibana

Forward the Kibana service to the local machine:

```bash
kubectl port-forward -n logging \
  service/efk-kb-http 5601:5601
```

Open the following address in a browser:

`https://localhost:5601`

Retrieve the Elasticsearch login password:

```bash
kubectl get secret efk-es-elastic-user \
  -n logging \
  -o go-template='{{.data.elastic | base64decode}}{{"\n"}}'
```

Use the following credentials:

- **Username:** `elastic`
- **Password:** The value returned by the command.

Keep the password private. Do not expose Kibana or Elasticsearch directly to the public internet without appropriate authentication, network restrictions, and TLS configuration.

## 8. Verify Log Ingestion

### 8.1 Check Fluent Bit Logs

Inspect recent collector logs for errors:

```bash
kubectl logs -n logging daemonset/fluent-bit --since=10m \
  | grep -iE 'error|warn|retry|failed|401|403'
```

Resolve any persistent ingestion errors before validating the data in Kibana.

### 8.2 Inspect Elasticsearch Indices

Use port forwarding to access Elasticsearch locally:

```bash
kubectl port-forward -n logging \
  service/efk-es-http 9200:9200
```

In another terminal, set the Elasticsearch password and list the indices:

```bash
ES_PASSWORD=$(kubectl get secret efk-es-elastic-user \
  -n logging \
  -o go-template='{{.data.elastic | base64decode}}')

curl -sk -u "elastic:$ES_PASSWORD" \
  "https://localhost:9200/_cat/indices?v"
```

Confirm that the expected Kubernetes log indices exist and contain documents.

A yellow index health status can be expected in a single-node Elasticsearch deployment when replica shards cannot be allocated. Green health is preferable for production, where appropriate replicas and multiple nodes should be configured.

### 8.3 Explore Logs in Kibana

1. Open **Discover** in Kibana.
2. Create a data view matching the configured log index pattern, such as `kubernetes-*`.
3. Select `@timestamp` as the time field when prompted.
4. Set the time range to **Last 15 minutes**.
5. Search for application logs and inspect their Kubernetes metadata.

If logs do not appear, verify the actual index prefix, the selected time range, and the field names present in the indexed documents.

## 9. Validate with Test Application Logs

Deploy a test workload that periodically generates application logs at different severity levels, such as `INFO`, `WARN`, and `ERROR`.

Verify the workload's output:

```bash
kubectl logs -n demo-argo log-generator --tail=10
```

Then search for the workload in Kibana. For example, if the Kubernetes metadata field is available:

```text
kubernetes.pod_name : "log-generator"
```

For structured JSON logs, use consistent fields such as `level`, `service`, `status_code`, and `message`. Confirm the actual indexed field paths before creating queries.

## 10. Build a Logging Dashboard

Use Kibana Lens or dashboard visualizations to create a basic application logging dashboard.

Recommended visualizations include:

- **Log volume over time:** Identify increases or sudden spikes in application activity.
- **Logs by severity:** Compare `INFO`, `WARN`, and `ERROR` events.
- **Errors by service:** Identify services generating the most errors.
- **HTTP status codes:** Track failed requests, including HTTP 500 responses.
- **Logs by Kubernetes workload:** Identify pods producing warnings or errors.

Save the dashboard so that it can be reused during troubleshooting and project demonstrations.

## 11. Troubleshooting

| Issue | What to investigate |
|---|---|
| Elasticsearch remains Pending | Node capacity, persistent volumes, and storage class configuration |
| Kibana is unavailable | Pod readiness, service configuration, port forwarding, and TLS |
| Fluent Bit cannot authenticate | Secret contents, environment variable configuration, and Elasticsearch credentials |
| Elasticsearch rejects documents | Mapping conflicts, field types, and Fluent Bit output configuration |
| No logs appear in Kibana | Fluent Bit input paths, collector errors, index patterns, and time filters |
| Elasticsearch reports yellow health | Replica allocation in a single-node cluster |
| Fluent Bit reports too many open files | File descriptor limits, inotify limits, and the number of watched container log files |

## 12. Security and Operational Considerations

- Store credentials in Kubernetes Secrets and restrict access through RBAC.
- Enable Elasticsearch certificate verification outside temporary lab environments.
- Monitor disk usage, memory utilization, and log ingestion volume.
- Configure retention policies to prevent indefinite log storage.
- Avoid collecting sensitive information, such as passwords, access tokens, or personal data, in application logs.
- Use multiple Elasticsearch nodes and appropriate persistent storage for production deployments.
- Review the Fluent Bit configuration and index mappings when introducing new log formats.

## 13. Existing Observability Stack

If Prometheus, Grafana, Loki, or Alloy are already installed in the cluster, keep them running while validating EFK.

Prometheus provides metrics, while Elasticsearch and Kibana provide centralized log storage and analysis. Loki and Alloy can also support log collection and exploration, so maintaining both logging backends may be unnecessary unless the project specifically requires a comparison or separate use cases.

## 14. Expected Outcome

After completing this setup, the Kubernetes environment should support:

- Centralized collection of application container logs.
- Storage and indexing of logs in Elasticsearch.
- Searching and filtering logs through Kibana.
- Investigation of application errors and HTTP failures.
- Visualization of log volume, severity, and service-level error patterns.

This implementation satisfies the centralized log aggregation requirement by complementing Prometheus-based metrics monitoring with a dedicated logging pipeline.
