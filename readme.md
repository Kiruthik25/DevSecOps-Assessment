# DevSecOps & Site Reliability Engineering (SRE) Assessment

## Overview

This project demonstrates an end-to-end DevOps workflow for deploying and managing a Python microservice with a known memory leak. It combines secure CI/CD practices, containerization, GitOps-based deployment, Kubernetes security controls, monitoring, centralized logging, and alerting.

The primary objective is to automate application delivery, deploy multiple application versions behind a shared Kubernetes Service, and detect application failures caused by excessive memory consumption.

The environment uses open-source tools running on a local Kubernetes cluster hosted on an EC2 instance.

## Architecture

Developer → GitHub Pull Request → Secure CI Pipeline → Image Build and Security Scanning → Container Registry → GitOps Manifest Update → Argo CD → Kubernetes Deployments → Prometheus, Grafana, and Centralized Logging

## What I Implemented

### 1. Containerization and Secure CI/CD

- Containerized the Python application using Docker.
- Established a pull-request-based workflow for validation before merging changes into the main branch.
- Integrated code linting, unit testing, static application security testing, and container image vulnerability scanning.
- Automated image publishing and Kubernetes manifest updates following successful releases.

### 2. GitOps Deployment with Argo CD

- Configured Argo CD to synchronize Kubernetes resources from Git.
- Deployed three application versions using separate Kubernetes Deployments.
- Configured a shared Kubernetes Service to distribute traffic across the application instances.
- Applied memory requests and a `400Mi` memory limit to help contain the application's memory leak and demonstrate Kubernetes OOM handling.

### 3. Kubernetes Security and Resource Management

Implemented Kubernetes configuration for:

- Namespace isolation and workload organization.
- ConfigMaps and Secrets for configuration management.
- ServiceAccounts and RBAC for access control.
- NetworkPolicies for controlling pod communication.
- ResourceQuotas and LimitRanges for resource governance.
- Init containers for initialization tasks.

### 4. Monitoring and Observability

- Deployed Prometheus and Grafana for infrastructure and application monitoring.
- Configured dashboards to observe node CPU, memory, disk usage, and pod-level resource consumption.
- Implemented centralized application log collection and analysis using the EFK stack.
- Configured alerting for application health failures and memory-related container termination events.

### 5. Failure Detection and Troubleshooting

The application intentionally consumes increasing amounts of memory. Kubernetes memory limits provide a controlled way to demonstrate container termination due to memory exhaustion.

Prometheus and Grafana provide visibility into resource usage and workload health, while centralized logging helps investigate application behavior and failures.

The overall workflow demonstrates how CI/CD, GitOps, Kubernetes resource controls, monitoring, logging, and alerting work together to support reliable application operations.

## Repository Structure

The repository contains Kubernetes manifests and Helm values files for application deployment, configuration, security, resource management, and observability.

| File or group | Purpose |
|---|---|
| `deployment-v1.yaml`, `deployment-v2.yaml`, `deployment-v3.yaml` | Define the three application Deployments |
| `service.yaml` | Defines the shared application Service |
| `namespace.yaml` | Defines the Kubernetes namespace |
| `configmap.yaml` | Stores application configuration |
| `serviceaccount.yaml`, `rbac.yaml` | Define workload identity and access permissions |
| `networkpolicy.yaml` | Defines pod network communication rules |
| `resourcequota.yaml`, `limitrange.yaml` | Configure resource governance |
| `kustomization.yaml` | Organizes Kubernetes resources for deployment |
| `altering-rule.yaml` | Defines alerting rules |
| `grafana-smtp-values.yaml` | Configures Grafana SMTP-related settings |
| `routing-values.yaml` | Contains routing-related Helm configuration |
| `log-generator.yaml` | Defines a workload for generating test logs |

## End-to-End Workflow

1. A developer opens a pull request against the main branch.
2. The CI pipeline builds and validates the application, runs tests, and performs security checks.
3. After approval and merging, the release pipeline builds and publishes the container image.
4. The Kubernetes deployment manifests are updated with the released image tags.
5. Argo CD detects the Git changes and synchronizes the desired state with the Kubernetes cluster.
6. Kubernetes runs the application versions behind a shared Service and enforces their resource limits.
7. Prometheus collects metrics, Grafana visualizes performance and evaluates alerts, and the logging stack centralizes application logs.
8. When the application becomes unhealthy or exceeds its memory limit, the monitoring and alerting setup helps identify and investigate the failure.

## Outcome

This project demonstrates practical experience with secure delivery pipelines, containerization, GitOps, Kubernetes security and resource management, infrastructure monitoring, centralized logging, and operational troubleshooting.

It provides an end-to-end example of automating application delivery and observing a deliberately unstable microservice in a controlled Kubernetes environment.
