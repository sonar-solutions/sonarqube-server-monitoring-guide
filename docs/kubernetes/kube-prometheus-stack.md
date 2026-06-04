# Monitoring with kube-prometheus-stack

This guide explains how to set up Prometheus scraping for SonarQube Server when using the [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) Helm chart, which deploys Prometheus via the [Prometheus Operator](https://prometheus-operator.dev/).

## Helm values

The SonarQube Helm chart can generate the `PodMonitor` for you. Enable it in your `values.yaml`:

```yaml
prometheusMonitoring:
  podMonitor:
    enabled: true
    labels:
      release: prometheus
```

The `release: prometheus` label is important: it is how the Prometheus Operator deployed by kube-prometheus-stack knows to pick up the `PodMonitor`. By default, the Operator only selects monitors carrying the release label of its own Helm installation — adjust the value to match your kube-prometheus-stack release name if it differs.

## Verifying that metrics are being scraped

After applying the Helm values, verify that the scrape targets are active in Prometheus.

### Check that the PodMonitor was created

```bash
kubectl get podmonitor -n <podmonitor-namespace>
```

You should see a `PodMonitor` resource for SonarQube in the output.

### Check the Prometheus targets UI

Open the Prometheus UI and navigate to **Status → Targets**. Look for targets corresponding to the SonarQube pod. All three endpoints should appear with a `UP` state:

- The `/api/monitoring/metrics` endpoint
- The web process JMX exporter port
- The CE process JMX exporter port

### Confirm metrics are arriving

Run a quick query in the Prometheus UI or Grafana to confirm data is flowing. For example, query for `sonarqube_web_uptime_minutes`. If the query returns a result, the SonarQube Prometheus endpoint is being scraped successfully.

## Next steps

- Set up the [Grafana dashboard](../../dashboards/sonarqube-grafana-prometheus-k8s/) to visualize the collected metrics
- Review the [Prometheus alert catalogue](../prometheus-queries.md) for recommended alert conditions
