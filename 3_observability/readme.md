# 📊 Observability Platform (OpenTelemetry + New Relic)

## 📌 Overview

This project implements a **production-grade observability solution** for a Kubernetes platform using:

- **OpenTelemetry (OTel)** → instrumentation + collection  
- **OpenTelemetry Collector** → processing & routing  
- **New Relic (OTLP ingest)** → monitoring, dashboards, alerting  

---

## 🎯 Features

- ✅ Pipeline performance monitoring (CI/CD)
- ✅ Kubernetes resource utilization monitoring
- ✅ Tenant-level observability
- ✅ Logs ↔ Traces ↔ Metrics correlation
- ✅ Structured telemetry for NRQL queries
- ✅ Production-ready alerting

---

## 🏗️ Architecture
Applications / CI-CD / Services
│
▼
OpenTelemetry SDK / Auto Instrumentation
│
▼
OTel Gateway Collector (Deployment)
│
▼
New Relic (OTLP HTTP)
│
├── Traces
├── Metrics
├── Logs
└── Alerts

Kubernetes Nodes
│
▼
OTel Agent Collector (DaemonSet)
├── Kubelet metrics
├── Prometheus scrape
└── Pod logs



---

## ⚙️ Prerequisites

- Kubernetes (EKS / K8s)
- Helm
- kubectl
- New Relic License Key

---

# 🚀 Setup

## 1. Create Namespace

```bash
kubectl create namespace observability


2. Create New Relic Secret
kubectl -n observability create secret generic newrelic-secret \
  --from-literal=NEW_RELIC_LICENSE_KEY='<YOUR_KEY>'

3. Install OpenTelemetry Operator
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo update

helm upgrade --install opentelemetry-operator \
  open-telemetry/opentelemetry-operator \
  -n observability

Collectors
🔹 Gateway Collector (App + Pipeline)
apiVersion: opentelemetry.io/v1beta1
kind: OpenTelemetryCollector
metadata:
  name: otel-gateway
  namespace: observability
spec:
  mode: deployment
  replicas: 2

  env:
    - name: NEW_RELIC_LICENSE_KEY
      valueFrom:
        secretKeyRef:
          name: newrelic-secret
          key: NEW_RELIC_LICENSE_KEY

  config:
    receivers:
      otlp:
        protocols:
          grpc: {}
          http: {}

    processors:
      memory_limiter:
        limit_mib: 512
      batch:
        timeout: 10s

      resource:
        attributes:
          - key: deployment.environment
            value: prod
            action: upsert
          - key: k8s.cluster.name
            value: eks-prod
            action: upsert

    exporters:
      otlphttp/newrelic:
        endpoint: https://otlp.nr-data.net:4318
        headers:
          api-key: ${NEW_RELIC_LICENSE_KEY}

    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [memory_limiter, batch, resource]
          exporters: [otlphttp/newrelic]

        metrics:
          receivers: [otlp]
          processors: [memory_limiter, batch, resource]
          exporters: [otlphttp/newrelic]

        logs:
          receivers: [otlp]
          processors: [memory_limiter, batch, resource]
          exporters: [otlphttp/newrelic]
🔹 Agent Collector (Infra + Logs)
apiVersion: opentelemetry.io/v1beta1
kind: OpenTelemetryCollector
metadata:
  name: otel-agent
  namespace: observability
spec:
  mode: daemonset

  env:
    - name: NEW_RELIC_LICENSE_KEY
      valueFrom:
        secretKeyRef:
          name: newrelic-secret
          key: NEW_RELIC_LICENSE_KEY

  config:
    receivers:
      kubeletstats:
        collection_interval: 20s

      prometheus:
        config:
          scrape_configs:
            - job_name: kubernetes-pods
              kubernetes_sd_configs:
                - role: pod

      filelog:
        include:
          - /var/log/pods/*/*/*.log

    processors:
      memory_limiter:
        limit_mib: 256
      batch: {}

    exporters:
      otlphttp/newrelic:
        endpoint: https://otlp.nr-data.net:4318
        headers:
          api-key: ${NEW_RELIC_LICENSE_KEY}

    service:
      pipelines:
        metrics:
          receivers: [kubeletstats, prometheus]
          processors: [memory_limiter, batch]
          exporters: [otlphttp/newrelic]

        logs:
          receivers: [filelog]
          processors: [memory_limiter, batch]
          exporters: [otlphttp/newrelic]
🧪 Application Instrumentation
Instrumentation CR
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: platform-instrumentation
  namespace: observability
spec:
  exporter:
    endpoint: http://otel-gateway:4318

  sampler:
    type: parentbased_traceidratio
    argument: "0.2"

  env:
    - name: OTEL_RESOURCE_ATTRIBUTES
      value: service.name=myapp,deployment.environment=prod
Enable in Deployment
metadata:
  annotations:
    instrumentation.opentelemetry.io/inject-nodejs: "observability/platform-instrumentation"


🔧 CI/CD Pipeline Instrumentation
#!/bin/bash
START=$(date +%s)
echo "Running pipeline..."
sleep 5
END=$(date +%s)
DURATION=$((END-START))
echo "Pipeline duration: $DURATION"

###############
📊 Metrics Model
Core Metrics
platform_pipeline_runs_total
platform_pipeline_duration_seconds
platform_requests_total
platform_errors_total
platform_queue_depth
Attributes
tenant.id
service.name
pipeline.name
pipeline.stage
deployment.environment
k8s.cluster.name

########################
📊 Dashboards
Pipeline
Deployment success rate
Latency (p95)
Failures
Platform
CPU / Memory
Pod restarts
Node health
Tenant
Requests per tenant
Errors per tenant
Latency per tenant

#############################
📈 NRQL Queries
Pipeline Failures
SELECT count(*) 
FROM Metric 
WHERE metricName = 'platform_pipeline_failures_total'
FACET pipeline.name
CPU Usage
SELECT average(k8s.container.cpuUsedCores)
FROM Metric
FACET k8s.namespace.name
Errors per Tenant
SELECT rate(sum(platform_errors_total), 1 minute)
FROM Metric
FACET tenant.id    