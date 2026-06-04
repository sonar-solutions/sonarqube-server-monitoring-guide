# Kubernetes monitoring setup

This section covers setting up Prometheus-based monitoring for SonarQube Server deployed on Kubernetes via the [official Helm chart](https://github.com/SonarSource/helm-chart-sonarqube).

## Prerequisites

Before setting up monitoring, the following must be in place.

### JMX exporter enabled

The SonarQube Prometheus metrics endpoint is always available, but the JVM metrics for the web and Compute Engine processes are exposed by a JMX exporter that is bundled in the Helm chart and disabled by default. Enable it in your `values.yaml`:

```yaml
# Community Build, Developer, and Enterprise editions
prometheusExporter:
  enabled: true
```

```yaml
# Data Center Edition
applicationNodes:
  prometheusExporter:
    enabled: true
```

That is all that is required to expose the JVM metrics. Optionally, you can pin a more recent exporter version via the same `prometheusExporter` block — see the [chart's values reference](https://github.com/SonarSource/helm-chart-sonarqube) for the available options.

### System passcode configured

The SonarQube Prometheus endpoint (`/api/monitoring/metrics`) requires the system passcode, set via `sonar.web.systemPasscode`. This is not specific to monitoring: the passcode is part of any SonarQube Server deployment (it backs the readiness probe, among other things), so a running Helm release already has it configured.

The setup guides reference this existing passcode in the Prometheus scrape configuration to authenticate requests to the metrics endpoint.

### Prometheus and the Prometheus Operator running in the cluster

The supported setups rely on a Prometheus instance managed by the [Prometheus Operator](https://github.com/prometheus-operator/prometheus-operator) running in your cluster:

- **kube-prometheus-stack** — the [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) Helm chart, which deploys Prometheus and the Operator together
- **Azure managed Prometheus** — Azure Monitor managed Prometheus on AKS, which runs an Operator-compatible collector

## How scraping works: the Prometheus Operator and PodMonitor

Both supported setups use the same mechanism, so it is worth understanding before you pick a guide.

Rather than editing a central Prometheus configuration file, the [Prometheus Operator](https://github.com/prometheus-operator/prometheus-operator) discovers scrape targets from Kubernetes custom resources. A **PodMonitor** is one such resource: it declaratively selects a set of pods (by label) and describes how to scrape them — which port, path, interval, and authentication to use. The Operator watches for `PodMonitor` objects and continuously translates them into the running Prometheus scrape configuration.

For SonarQube, a `PodMonitor` points Prometheus at all three metric sources — the `/api/monitoring/metrics` endpoint (supplying the system passcode as a bearer token) and the web and Compute Engine JMX exporter endpoints. Both kube-prometheus-stack and Azure managed Prometheus understand `PodMonitor` resources, but they rely on different CRDs. The Helm chart can natively generate a valid `PodMonitor` for kube-prometheus-stack, but not for Azure managed Prometheus — which is where the two setup guides diverge.

## Setup options

Choose the guide that matches your environment:

| Setup | Guide |
|-------|-------|
| kube-prometheus-stack (Prometheus Operator) | [kube-prometheus-stack setup](kube-prometheus-stack.md) |
| Azure managed Prometheus (AKS) | [Azure managed Prometheus setup](azure-managed-prometheus.md) |

### Other setups

Other Prometheus deployments — for example a standalone Prometheus installation, or a Prometheus that does not use the Operator — are not currently covered by a dedicated guide. The same principles still apply: expose the metrics endpoint, authenticate with the system passcode, and point a scrape job at it. Where the Operator-based guides rely on a `PodMonitor`, a non-Operator setup uses an equivalent scrape job in the Prometheus configuration. The kube-prometheus-stack guide is the closest starting point to adapt.

Contributions and/or suggestions that expand this guide to additional setups are welcome.
