# Datadog Agent — EKS Installation & Configuration Guide

> **Production-ready Datadog observability setup on Amazon EKS using the Datadog Operator with APM, log collection, NPM, and namespace-level filtering.**

[![EKS](https://img.shields.io/badge/EKS-1.31%2B-orange?logo=amazon-aws)](https://docs.aws.amazon.com/eks/)
[![Datadog Agent](https://img.shields.io/badge/Datadog%20Agent-v7.x-purple?logo=datadog)](https://docs.datadoghq.com/)
[![Helm](https://img.shields.io/badge/Helm-v3.x-blue?logo=helm)](https://helm.sh/)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

---

## Overview

This repository provides a complete, step-by-step guide and all required configuration files to deploy the Datadog Agent on an Amazon EKS cluster using the **Datadog Operator** pattern. It covers everything from initial prerequisites to production-grade upgrades and rollback procedures.

| Detail | Value |
|---|---|
| EKS Version | 1.31 (upgrading to 1.34) |
| Datadog Agent | v7.x (latest stable) |
| Operator Version | v1.x (latest stable) |
| Installation Namespace | `datadog` |
| Monitored Namespaces | `dev-env-ns`, `qa-env-ns` |
| Features Enabled | APM, Metrics, Logs, NPM |

---

## Features

- **Datadog Operator** — Kubernetes-native lifecycle management of the Datadog Agent
- **APM with UDS** — Low-latency Application Performance Monitoring via Unix Domain Socket
- **Full Log Collection** — Container log collection with file-based tailing
- **Namespace Filtering** — Strict include/exclude rules; zero data leakage from unmonitored namespaces
- **Network Performance Monitoring (NPM)** — Layer 4 traffic visibility across the cluster
- **Kubernetes State Metrics** — Pod, deployment, and node health metrics
- **HA Cluster Agent** — 2-replica Cluster Agent for production resilience
- **Auto-Instrumentation** — Admission controller support for Java, Node.js, Python, and .NET
- **Upgrade & Rollback Procedures** — Documented Helm-based upgrade path including EKS 1.31 → 1.34

---

## Repository Structure

```
.
├── README.md
├── datadog-agent.yaml          # DatadogAgent Custom Resource (primary config)
├── docs/
│   └── DATADOG_AGENT_Installation_and_Configuration_Guide.pdf
└── scripts/
    ├── 01-create-namespace.sh  # Namespace and secret setup
    ├── 02-install-operator.sh  # Helm operator install
    ├── 03-apply-agent.sh       # Apply DatadogAgent CR
    └── 04-verify.sh            # Verification commands
```

---

## Prerequisites

| Tool | Version |
|---|---|
| `kubectl` | Compatible with EKS 1.31+ |
| `helm` | v3.x |
| `aws cli` | v2.x with EKS permissions |
| `eksctl` | Latest (optional) |
| `kubeconfig` | Configured for target cluster |

---

## Quick Start

### 1. Verify Cluster Connectivity

```bash
kubectl config current-context
kubectl get nodes
```

### 2. Create Namespace & Store API Keys as Secrets

```bash
kubectl create namespace datadog

kubectl create secret generic datadog-secret \
  --from-literal api-key=<YOUR_API_KEY> \
  --from-literal app-key=<YOUR_APP_KEY> \
  --namespace datadog
```

> **Security Note:** Never hardcode API keys in YAML files. Always use Kubernetes Secrets.

### 3. Install the Datadog Operator via Helm

```bash
helm repo add datadog https://helm.datadoghq.com
helm repo update

helm install datadog-operator datadog/datadog-operator \
  --namespace datadog \
  --create-namespace \
  --set replicaCount=1

# Verify operator is running
kubectl get pods -n datadog -l app.kubernetes.io/name=datadog-operator
```

### 4. Apply the DatadogAgent Custom Resource

```bash
kubectl apply -f datadog-agent.yaml

# Watch reconciliation
kubectl get datadogagent -n datadog -w

# Verify DaemonSet (one pod per node)
kubectl get pods -n datadog -l agent.datadoghq.com/name=datadog
```

### 5. Verify APM

```bash
AGENT_POD=$(kubectl get pods -n datadog \
  -l agent.datadoghq.com/component=agent \
  -o jsonpath="{.items[0].metadata.name}")

kubectl exec -it $AGENT_POD -n datadog -- agent status | grep -A 10 "APM Agent"
# Expected: Status: Running
```

---

## Namespace Filtering

Only `dev-env-ns` and `qa-env-ns` are monitored. All other namespaces are excluded via the following environment variable pattern in `datadog-agent.yaml`:

```yaml
- name: DD_CONTAINER_INCLUDE
  value: "kube_namespace:dev-env-ns kube_namespace:qa-env-ns"
- name: DD_CONTAINER_EXCLUDE
  value: "kube_namespace:.*"
```

> The `INCLUDE` directive takes precedence over `EXCLUDE` when both match, ensuring zero data leakage from other namespaces.

---

## APM Auto-Instrumentation

Add the following annotations to your application Deployment manifests in monitored namespaces:

```yaml
annotations:
  admission.datadoghq.com/enabled: "true"
  admission.datadoghq.com/java-lib.version: "latest"    # Java
  admission.datadoghq.com/js-lib.version: "latest"      # Node.js
  admission.datadoghq.com/python-lib.version: "latest"  # Python
  admission.datadoghq.com/dotnet-lib.version: "latest"  # .NET
```

To patch existing deployments in bulk:

```bash
for deploy in app-server-deployment app1-server-deployment app3-server-deployment; do
  kubectl patch deployment $deploy -n dev-env-ns \
    --type=merge \
    -p '{"spec":{"template":{"metadata":{"annotations":{"admission.datadoghq.com/enabled":"true","admission.datadoghq.com/java-lib.version":"latest"}}}}}'
done
```

---

## Upgrade Procedure

### Upgrade Datadog Operator

```bash
helm repo update
helm upgrade datadog-operator datadog/datadog-operator \
  --namespace datadog \
  --reuse-values

kubectl rollout status deployment/datadog-operator -n datadog
```

### Upgrade Datadog Agent

Edit `datadog-agent.yaml` to pin the new image tag, then re-apply:

```yaml
override:
  nodeAgent:
    image:
      tag: "7.56.0"
  clusterAgent:
    image:
      tag: "7.56.0"
```

```bash
kubectl apply -f datadog-agent.yaml
kubectl rollout status daemonset/datadog-agent -n datadog
```

### EKS Cluster Upgrade (1.31 → 1.34)

> EKS requires upgrading **one minor version at a time**. Plan for 3 separate upgrade windows (~15–20 min each).

```
1.31 → 1.32 → 1.33 → 1.34
```

Always upgrade the Datadog Operator **before** upgrading the EKS control plane.

---

## Rollback

```bash
helm history datadog-operator -n datadog
helm rollback datadog-operator -n datadog
```

---

## Troubleshooting

| Issue | Resolution |
|---|---|
| Agent pods in `CrashLoopBackOff` | Check `kubectl logs <pod> -n datadog` — usually an invalid API key |
| No data in Datadog dashboard | Run `agent status` inside the pod; verify outbound 443 to `datadoghq.com` is open |
| APM traces not appearing | Confirm app is sending to `status.hostIP:8126` or UDS socket |
| Unwanted namespaces appearing | Re-check `DD_CONTAINER_EXCLUDE` — regex `.*` must be set before INCLUDE |
| Operator not reconciling | `kubectl rollout restart deployment/datadog-operator -n datadog` |
| High memory usage | Reduce `containerCollectAll`, tune resource limits in the CR |

### Useful Diagnostic Commands

```bash
# Full agent status
kubectl exec -it $AGENT_POD -n datadog -- agent status

# Confirm namespace filtering is active
kubectl exec -it $AGENT_POD -n datadog -- agent config | grep -A 2 "container_include"

# Confirm env vars are set
kubectl exec -it $AGENT_POD -n datadog -- env | grep DD_CONTAINER

# Live logs
kubectl logs -f $AGENT_POD -n datadog
```

---

## Validation Checklist

- [ ] `kubectl get pods -n datadog` — all pods `Running`
- [ ] DaemonSet shows one pod per node
- [ ] Cluster Agent shows `2/2 Ready`
- [ ] APM status: `Running` with UDS socket active
- [ ] Datadog dashboard → Infrastructure → Containers shows only `dev-env-ns` and `qa-env-ns`
- [ ] APM → Services — instrumented services visible within 5–10 minutes
- [ ] Logs Agent sending to `agent-httpintake.logs.us5.datadoghq.com` on port 443

---

## References

- [Datadog Operator Docs](https://docs.datadoghq.com/containers/datadog_operator/)
- [Kubernetes APM Setup](https://docs.datadoghq.com/tracing/trace_collection/automatic_instrumentation/)
- [Amazon EKS Integration](https://docs.datadoghq.com/integrations/amazon_eks/)

---

## Author

**Satish Khair** — [github.com/satishkhair07](https://github.com/satishkhair07)
