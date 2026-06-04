# What to monitor

This page is an **orientation**: it explains where SonarQube Server metrics come from, what each source exposes, and which signals matter. It does *not* explain how to read or act on each signal — that lives in the dashboard's [how-to guide](../dashboards/sonarqube-grafana-prometheus-k8s/how-to-use-this-dashboard.md), which is the single source of truth for operational interpretation. The [alert catalogue](prometheus-queries.md) holds the PromQL and thresholds.

## Metrics sources at a glance

A complete setup draws from four sources. The first three come from SonarQube itself; the fourth comes from your Kubernetes monitoring stack.

| Source | What it covers | Scrape target |
|--------|---------------|---------------|
| [SonarQube Prometheus endpoint](#1-sonarqube-prometheus-endpoint) | Application-level health and SonarQube-specific signals | `/api/monitoring/metrics` on the SonarQube pod |
| [JMX exporter — web process](#2-jmx-prometheus-exporter--web-process) | JVM metrics for the web process | JMX exporter port (`endpoint="monitoring-web"`) |
| [JMX exporter — Compute Engine process](#3-jmx-prometheus-exporter--compute-engine-process) | JVM metrics + SonarQube JMX MBeans for the CE process | JMX exporter port (`endpoint="monitoring-ce"`) |
| [Kubernetes platform metrics](#4-kubernetes-platform-metrics) | Pod health, replicas, CPU, memory | kube-state-metrics + cAdvisor/kubelet |

### Metric naming conventions

The three naming styles you will see are a reliable way to tell sources apart:

| Naming | Source | Example |
|--------|--------|---------|
| `sonarqube_snake_case` | SonarQube Prometheus endpoint (Web API) | `sonarqube_compute_engine_pending_tasks_total` |
| `SonarQube_PascalCase` | SonarQube JMX MBeans (CE process) | `SonarQube_ComputeEngineTasks_LongestTimePending` |
| `java_lang_*` | Standard JVM metrics via JMX | `java_lang_Memory_HeapMemoryUsage_committed` |

> [!NOTE]
> JMX-exported metrics appear as **Untyped** in Prometheus, because JMX MBeans carry no Prometheus type metadata. Some signals (such as the CE pending-task queue) are available from *both* the native endpoint and JMX — the [alert catalogue](prometheus-queries.md) specifies which one each alert uses.


## 1. SonarQube Prometheus endpoint

**Endpoint**: `GET /api/monitoring/metrics`  
**Format**: [Prometheus / OpenMetrics text exposition format](https://prometheus.io/docs/instrumenting/exposition_formats/)  
**Authentication**: HTTP Bearer token (or `X-Sonar-Passcode` header) — the value of `sonar.web.systemPasscode`  
**Availability**: All deployment types (bare metal, Docker, Kubernetes)

The primary source for SonarQube-specific operational signals. Implemented directly in SonarQube Server; no additional agents or sidecars required.

**Exposes:** application health, Compute Engine queue/workers/tasks, license (LOC utilization and expiry), Elasticsearch health and storage, DevOps platform integration status, and IDE connection count.

## 2. JMX Prometheus exporter — web process

**Source**: [JMX Prometheus Java agent](https://github.com/prometheus/jmx_exporter) attached to the SonarQube web process  
**Configuration**: Native in the SonarQube Helm chart — no sidecar or manual agent install

Standard JVM metrics for the **web process**, which handles all HTTP traffic (UI, API, scanner and IDE connections).

## 3. JMX Prometheus exporter — Compute Engine process

**Source**: JMX Prometheus Java agent attached to the SonarQube Compute Engine (CE) process  
**Configuration**: Native in the SonarQube Helm chart

Standard JVM metrics for the **CE process** (which processes analysis tasks), **plus** SonarQube-specific CE counters registered as JMX MBeans (`SonarQube_ComputeEngineTasks_*`). The CE process is long-lived and memory-intensive, so its heap and task counters are central operational signals.

## 4. Kubernetes platform metrics

**Source**: [kube-state-metrics](https://github.com/kubernetes/kube-state-metrics) (`kube_pod_*`, `kube_deployment_*`, `kube_statefulset_*`) and cAdvisor/kubelet (`container_cpu_*`, `container_memory_working_set_bytes`), plus Prometheus's own `scrape_duration_seconds`  
**Configuration**: Provided by the monitoring stack, **not** by SonarQube. Bundled in kube-prometheus-stack by default; must be enabled for Azure managed Prometheus.

Infrastructure-level signals: pod readiness (desired vs ready), pod uptime, container CPU (request/limit/usage and throttling), and container memory (request/limit/working-set). This is the source a future VM dashboard would replace with VM/host metrics.

## 5. Database

Database monitoring is out of scope for this guide. The signals described above cover everything SonarQube itself exposes; database-tier metrics depend on the specific engine (PostgreSQL, SQL Server, Oracle) and how it is deployed, and are not provided by SonarQube. See the [Database section](../dashboards/sonarqube-grafana-prometheus-k8s/how-to-use-this-dashboard.md#database) of the dashboard how-to for context on why it matters and what to track separately.

## Signal map

The operationally important signals, each linked to where it is interpreted (dashboard how-to) and alerted (alert catalogue).

| Signal | Source | How to read it | Alert |
|--------|--------|----------------|-------|
| Instance up / down | 1 (metric absence) | [Quick Glance](../dashboards/sonarqube-grafana-prometheus-k8s/how-to-use-this-dashboard.md#quick-glance) | [Availability](prometheus-queries.md#availability) |
| License — LOC utilization | 1 | [Quick Glance](../dashboards/sonarqube-grafana-prometheus-k8s/how-to-use-this-dashboard.md#quick-glance) | [License](prometheus-queries.md#license) |
| License — days to expiry | 1 | [Quick Glance](../dashboards/sonarqube-grafana-prometheus-k8s/how-to-use-this-dashboard.md#quick-glance) | [License](prometheus-queries.md#license) |
| IDE connections | 1 | [Quick Glance](../dashboards/sonarqube-grafana-prometheus-k8s/how-to-use-this-dashboard.md#quick-glance) | informational |
| CE queue depth (pending) | 1 / 3 | [Compute Engine](../dashboards/sonarqube-grafana-prometheus-k8s/how-to-use-this-dashboard.md#compute-engine-statistics--background-tasks) | [Compute Engine](prometheus-queries.md#compute-engine) |
| CE max pending time | 3 | [Compute Engine](../dashboards/sonarqube-grafana-prometheus-k8s/how-to-use-this-dashboard.md#compute-engine-statistics--background-tasks) | [Compute Engine](prometheus-queries.md#compute-engine) |
| CE task failure rate | 3 | [Compute Engine](../dashboards/sonarqube-grafana-prometheus-k8s/how-to-use-this-dashboard.md#compute-engine-statistics--background-tasks) | [Compute Engine](prometheus-queries.md#compute-engine) |
| Elasticsearch health | 1 | [Status](../dashboards/sonarqube-grafana-prometheus-k8s/how-to-use-this-dashboard.md#status) | [Elasticsearch](prometheus-queries.md#elasticsearch) |
| Elasticsearch storage | 1 | [Status](../dashboards/sonarqube-grafana-prometheus-k8s/how-to-use-this-dashboard.md#status) | [Elasticsearch](prometheus-queries.md#elasticsearch) |
| DevOps integration status | 1 | [Status](../dashboards/sonarqube-grafana-prometheus-k8s/how-to-use-this-dashboard.md#status) | [DevOps integrations](prometheus-queries.md#devops-platform-integrations) |
| JVM heap — web | 2 | [JVM Metrics](../dashboards/sonarqube-grafana-prometheus-k8s/how-to-use-this-dashboard.md#sonarqube-web-and-compute-engine-jvm-metrics) | [JVM heap](prometheus-queries.md#jvm-heap) |
| JVM heap — CE | 3 | [JVM Metrics](../dashboards/sonarqube-grafana-prometheus-k8s/how-to-use-this-dashboard.md#sonarqube-web-and-compute-engine-jvm-metrics) | [JVM heap](prometheus-queries.md#jvm-heap) |
| Scrape duration | 4 | [Scraping Performance](../dashboards/sonarqube-grafana-prometheus-k8s/how-to-use-this-dashboard.md#scraping-performance) | [Optional](prometheus-queries.md#optional-scrape-duration) |
| Pod readiness | 4 | [Kubernetes Metrics](../dashboards/sonarqube-grafana-prometheus-k8s/how-to-use-this-dashboard.md#kubernetes-metrics) | [Kubernetes platform](prometheus-queries.md#kubernetes-platform) |
| Container memory | 4 | [Kubernetes Metrics](../dashboards/sonarqube-grafana-prometheus-k8s/how-to-use-this-dashboard.md#kubernetes-metrics) | [Kubernetes platform](prometheus-queries.md#kubernetes-platform) |
| Database tier | external | [Database](../dashboards/sonarqube-grafana-prometheus-k8s/how-to-use-this-dashboard.md#database) | out of scope |
