## Monitoring Solution 

•	**Prometheus:** An open-source systems monitoring and alerting toolkit that collects metrics from configured targets at given intervals.

•	**Grafana:** A multi-platform open-source analytics and interactive visualization web application that provides charts, graphs, and alerts.

•	**Alertmanager:** A component of the Prometheus ecosystem that handles alerts sent by client applications.

### Step-by-Step Deployment Guide

### 1. Install the Prometheus

```bash
# Add the Prometheus community Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

# Update Helm repositories
helm repo update

# Install the Prometheus Operator using Helm
helm install prometheus-operator prometheus-community/kube-prometheus-stack
```

### 2.  Configure Prometheus to Scrape Metrics


Prometheus needs to be configured to scrape metrics from the nodes and services.
• **Node Metrics:**
•	The Node Exporter is typically deployed as a DaemonSet to collect node metrics.
•	The Prometheus Operator chart includes this by default.
• **Kubernetes Metrics:**
•	Deploy kube-state-metrics to collect metrics about Kubernetes API objects.
•	It’s included in the Prometheus Operator deployment.
• **Service Metrics:**
•	Services are exposing Prometheus metrics (usually at /metrics endpoint).
•	Annotate the Kubernetes services to be scraped by Prometheus:


```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
```

### 3. Access Grafana

```bash
# Forward the Grafana service to a local port
kubectl port-forward svc/prometheus-operator-grafana 3000:80
```

- Login Credentials:
  Username: admin
  Password: prom-operator

•	Import Dashboards:
    •	Pre-configured dashboards from Grafana’s dashboard repository, such as:
        •	Kubernetes cluster monitoring (ID: 315)
        •	Node exporter full (ID: 1860)

[Grafana Dashboards](https://grafana.com/grafana/dashboards/)


### Alert Definitions for 3 metrics

1.	High Memory Usage on Nodes
2.	Low Disk Space on Nodes
3.	Pods in CrashLoopBackOff State
---
1. High Memory Usage on Nodes:

```highmem.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: node-memory-usage-alert
  labels:
    alert: node-memory-usage
spec:
  groups:
    - name: node.rules
      rules:
        - alert: HighNodeMemoryUsage
          expr: (node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes * 100 > 80
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "High Memory Usage on Node {{ $labels.instance }}"
            description: "Memory usage is above 80% on node {{ $labels.instance }}. Current usage is {{ $value | printf "%.2f" }}%."
```

2.	Low Disk Space on Nodes:

```lowdisk.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: node-disk-space-alert
  labels:
    alert: node-disk-space
spec:
  groups:
    - name: node.rules
      rules:
        - alert: LowNodeDiskSpace
          expr: (node_filesystem_size_bytes{mountpoint="/"} - node_filesystem_free_bytes{mountpoint="/"}) / node_filesystem_size_bytes{mountpoint="/"} * 100 > 85
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "Low Disk Space on Node {{ $labels.instance }}"
            description: "Disk usage is above 85% on node {{ $labels.instance }}. Current usage is {{ $value | printf "%.2f" }}%."
```


3.	Pods in CrashLoopBackOff State:

```crash.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: pod-crashloopbackoff-alert
  labels:
    alert: pod-crashloopbackoff
spec:
  groups:
    - name: kubernetes.rules
      rules:
        - alert: PodCrashLoopBackOff
          expr: count(kube_pod_container_status_waiting_reason{reason="CrashLoopBackOff"}) > 0
          for: 2m
          labels:
            severity: critical
          annotations:
            summary: "Pod in CrashLoopBackOff State"
            description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} is in CrashLoopBackOff state."
```

Combine all the alert rules into sinfle prometheus-rules.yaml

```yaml
kubectl apply -f prometheus-rules.yaml
```
