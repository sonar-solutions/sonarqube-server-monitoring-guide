# Prometheus alert catalogue

This page is the **alert reference** for SonarQube Server monitoring. Each entry gives a PromQL expression, a recommended threshold, and a minimum duration (`for`). Adapt them to Prometheus Alertmanager, Azure Monitor alert rules, or any other alerting platform.

The alerts are derived from the [Grafana dashboard](../dashboards/sonarqube-grafana-prometheus-k8s/sonarqube-grafana-prometheus-k8s-dashboard.json), and the recommended thresholds mirror the dashboard's own color thresholds where one exists. For what each signal *means* and how to act on it, follow the **how to use** link in each section — that interpretation is not repeated here.

> [!NOTE]
> The dashboard already contains every **display** query as a panel data source. This page is only about **alerting**. You do not need to copy display queries from here.

**Conventions used below:**
- Examples filter on `namespace="sonarqube"`. Replace it with your namespace, or remove the selector to evaluate across all namespaces.
- JVM and Kubernetes alerts are written **per pod** (no aggregation that hides individual pods), so an alert fires for the specific pod that breaches the threshold.
- Severities are a suggested two-tier split (`warning` / `critical`); collapse to one tier if your platform prefers.

---

## Availability

### Instance up / down

Fires when SonarQube stops reporting metrics (crash, OOM-kill, restart loop). Detected by *absence* of the web uptime metric rather than a health endpoint. → [how to use](../dashboards/sonarqube-grafana-prometheus-k8s/how-to-use-this-dashboard.md#quick-glance)

```promql
absent(sonarqube_web_uptime_minutes{namespace="sonarqube"})
```

| Severity | Condition | `for` |
|----------|-----------|-------|
| critical | `== 1` | 3m |

---

## License

### LOC utilization

Percentage of licensed Lines of Code consumed. At 100%, SonarQube stops accepting new analyses. → [how to use](../dashboards/sonarqube-grafana-prometheus-k8s/how-to-use-this-dashboard.md#quick-glance)

```promql
max without (pod, instance) (
  sonarqube_license_number_of_lines_analyzed_total{namespace="sonarqube"}
  /
  (
    sonarqube_license_number_of_lines_analyzed_total{namespace="sonarqube"}
    + sonarqube_license_number_of_lines_remaining_total{namespace="sonarqube"}
  )
)
```

| Severity | Condition | `for` |
|----------|-----------|-------|
| warning | `> 0.8` | 1h |
| critical | `> 0.9` | 1h |

### Days to license expiry

An expired license stops SonarQube from functioning. → [how to use](../dashboards/sonarqube-grafana-prometheus-k8s/how-to-use-this-dashboard.md#quick-glance)

```promql
min without (pod, instance) (
  sonarqube_license_days_before_expiration_total{namespace="sonarqube"}
)
```

| Severity | Condition | `for` |
|----------|-----------|-------|
| warning | `< 30` | 1h |
| critical | `< 7` | 1h |

---

## Compute Engine

> [!NOTE]
> CE queue depth and pending time have **no universal thresholds** — the right value depends on your instance size and typical load. Establish a baseline for your environment and set the thresholds below relative to it. → [how to use](../dashboards/sonarqube-grafana-prometheus-k8s/how-to-use-this-dashboard.md#compute-engine-statistics--background-tasks)

### Queue depth (pending tasks)

Tasks waiting because all CE workers are busy. A steadily growing queue means work is arriving faster than it can be processed.

```promql
avg without (pod, instance) (
  sonarqube_compute_engine_pending_tasks_total{namespace="sonarqube"}
)
```

| Severity | Condition | `for` |
|----------|-----------|-------|
| warning | `> <your baseline>` (e.g. `> 10`) | 3m |

### Max task pending time

How long the longest-waiting task has been queued (milliseconds). More sensitive than queue depth alone.

```promql
avg without (pod, instance) (
  SonarQube_ComputeEngineTasks_LongestTimePending{namespace="sonarqube"}
)
```

| Severity | Condition | `for` |
|----------|-----------|-------|
| warning | `> <typical analysis duration in ms>` (e.g., `180000`) | 3m |

### Task failure rate

Background-task failures are normally rare; a rising failure rate warrants investigation.

```promql
sum(increase(SonarQube_ComputeEngineTasks_ErrorCount{namespace="sonarqube"}[1h]))
/
clamp_min(
  sum(increase(SonarQube_ComputeEngineTasks_SuccessCount{namespace="sonarqube"}[1h]))
  + sum(increase(SonarQube_ComputeEngineTasks_ErrorCount{namespace="sonarqube"}[1h])),
  1
)
```

| Severity | Condition | `for` |
|----------|-----------|-------|
| warning | `> 0.05` (5% of tasks failing) | 15m |

---

## Elasticsearch

### Cluster health status

`0` means the embedded Elasticsearch cluster status is **RED** — primary shards unavailable; SonarQube will malfunction. → [how to use](../dashboards/sonarqube-grafana-prometheus-k8s/how-to-use-this-dashboard.md#status)

```promql
min(sonarqube_health_elasticsearch_status{namespace="sonarqube"})
```

| Severity | Condition | `for` |
|----------|-----------|-------|
| critical | `== 0` | 5m |

### Storage utilization

Elasticsearch switches to read-only near ~90% disk, taking SonarQube down. Alert below that to leave time to resize the volume. → [how to use](../dashboards/sonarqube-grafana-prometheus-k8s/how-to-use-this-dashboard.md#status)

```promql
max by (node_name) (
  (
    sonarqube_elasticsearch_disk_space_total_bytes{namespace="sonarqube"}
    - sonarqube_elasticsearch_disk_space_free_bytes{namespace="sonarqube"}
  )
  / sonarqube_elasticsearch_disk_space_total_bytes{namespace="sonarqube"}
)
```

| Severity | Condition | `for` |
|----------|-----------|-------|
| warning | `> 0.6` | 10m |
| critical | `> 0.85` | 10m |

---

## DevOps platform integrations

`0` means at least one connection for that platform is failing, which blocks pull request decoration. → [how to use](../dashboards/sonarqube-grafana-prometheus-k8s/how-to-use-this-dashboard.md#status)

> [!IMPORTANT]
> Unconfigured integrations also report `0`. **Only create alerts for the platforms you actually use** — pick the matching metric(s) below and drop the rest.

```promql
# GitHub (swap the suffix for gitlab / azuredevops / bitbucket as needed)
min without (pod, instance) (
  sonarqube_health_integration_github_status{namespace="sonarqube"}
)
```

| Severity | Condition | `for` |
|----------|-----------|-------|
| warning | `== 0` | 10m |

---

## JVM heap

Committed heap as a fraction of configured maximum heap, per pod. A sustained high ratio means the process is near its memory ceiling — expect GC pauses, then OOM. → [how to use](../dashboards/sonarqube-grafana-prometheus-k8s/how-to-use-this-dashboard.md#sonarqube-web-and-compute-engine-jvm-metrics)

### Web process

```promql
java_lang_Memory_HeapMemoryUsage_committed{namespace="sonarqube", endpoint="monitoring-web"}
/
java_lang_Memory_HeapMemoryUsage_max{namespace="sonarqube", endpoint="monitoring-web"}
```

| Severity | Condition | `for` |
|----------|-----------|-------|
| warning | `> 0.8` | 15m |
| critical | `> 0.9` | 10m |

### Compute Engine process

```promql
java_lang_Memory_HeapMemoryUsage_committed{namespace="sonarqube", endpoint="monitoring-ce"}
/
java_lang_Memory_HeapMemoryUsage_max{namespace="sonarqube", endpoint="monitoring-ce"}
```

| Severity | Condition | `for` |
|----------|-----------|-------|
| warning | `> 0.8` | 15m |
| critical | `> 0.9` | 10m |

---

## Kubernetes platform

### Pod readiness (desired vs ready)

A non-zero gap means pods are missing. Brief gaps during rolling deployments are normal — the long `for` window filters that out. → [how to use](../dashboards/sonarqube-grafana-prometheus-k8s/how-to-use-this-dashboard.md#kubernetes-metrics)

```promql
# Application pods (Data Center Edition)
(
  kube_deployment_spec_replicas{namespace="sonarqube"}
  - kube_deployment_status_replicas_ready{namespace="sonarqube"}
) >= 1

# Search / single pod (StatefulSet)
(
  kube_statefulset_status_replicas{namespace="sonarqube"}
  - kube_statefulset_status_replicas_ready{namespace="sonarqube"}
) >= 1
```

| Severity | Condition | `for` |
|----------|-----------|-------|
| warning | gap `>= 1` | 15m |

### Container memory vs limit

Hitting the container memory limit triggers an OOM-kill (unlike CPU, which only throttles). Alert before the limit. → [how to use](../dashboards/sonarqube-grafana-prometheus-k8s/how-to-use-this-dashboard.md#kubernetes-metrics)

```promql
sum by (pod) (
  container_memory_working_set_bytes{namespace="sonarqube", container!="", container!="POD"}
)
/
sum by (pod) (
  kube_pod_container_resource_limits{namespace="sonarqube", resource="memory"}
)
```

| Severity | Condition | `for` |
|----------|-----------|-------|
| warning | `> 0.75` | 10m |
| critical | `> 0.85` | 5m |

> [!NOTE]
> Container/pod label conventions vary between clusters. If this query returns no data, check that `container_memory_working_set_bytes` and `kube_pod_container_resource_limits` share matching `pod` labels in your setup, and adjust the selectors accordingly.

---

## Optional: scrape duration

Scrape duration is a secondary, trend-based signal rather than a fixed-threshold alarm — a slow web server produces slow scrapes. There is no universal threshold; alert only if it rises and stays elevated relative to your baseline. → [how to use](../dashboards/sonarqube-grafana-prometheus-k8s/how-to-use-this-dashboard.md#scraping-performance)

```promql
avg_over_time(scrape_duration_seconds{namespace="sonarqube"}[5m])
```

| Severity | Condition | `for` |
|----------|-----------|-------|
| warning | `> <multiple of your baseline>` | 15m |
