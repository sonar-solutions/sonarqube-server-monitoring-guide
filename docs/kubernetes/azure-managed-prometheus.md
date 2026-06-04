# Monitoring with Azure managed Prometheus

This guide explains how to set up Prometheus scraping for SonarQube Server on AKS using [Azure Monitor managed Prometheus](https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/prometheus-metrics-overview).

## Why the default Helm chart PodMonitor does not work

Azure managed Prometheus understands `PodMonitor` resources, but it watches a different CRD group than the Prometheus Operator: `azmonitoring.coreos.com/v1` instead of the standard `monitoring.coreos.com/v1`. The SonarQube Helm chart only generates a `PodMonitor` in the standard group, so the Azure Monitor metrics addon never picks it up.

To scrape SonarQube with Azure managed Prometheus, you therefore write the `PodMonitor` by hand using the Azure CRD group. Everything else works the same as it does for the Prometheus Operator — including Bearer-token authentication via the `bearerTokenSecret` field.

## Prerequisites

- AKS cluster with [Azure Monitor managed Prometheus enabled](https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/prometheus-metrics-enable)
- SonarQube deployed with JMX exporters enabled and the system passcode configured (see [Kubernetes prerequisites](README.md))

## Custom PodMonitor manifests

You can obtain the Azure-flavored `PodMonitor` two ways: generate it from the Helm chart (recommended — it matches your actual deployment), or adapt a static example by hand. Either way, the `PodMonitor` scrapes all three metrics sources — the SonarQube Prometheus endpoint (`/api/monitoring/metrics`) and the web and Compute Engine JMX exporters — from every SonarQube pod its selector matches. In a Data Center Edition cluster the selector covers all application nodes, so each is scraped.

### Generating the manifest from the Helm chart (recommended)

The Helm chart already ships a `PodMonitor` template. Render it with `helm template` and swap the CRD group for the Azure one. This produces a manifest whose labels, ports, namespace, and `bearerTokenSecret` wiring already match your deployment, so it usually needs no further editing.

- `helm template` renders the `PodMonitor` template using your configuration (namespace, values file, `--set` values).
- The CRD group `monitoring.coreos.com/v1` is replaced with `azmonitoring.coreos.com/v1` so the Azure Monitor metrics addon processes it.
- The result is written to `sonarqube-podmonitor.yaml`, ready to [apply](#applying-the-manifest).

> [!IMPORTANT]
> The `helm template` invocation must use the **same** release name, namespace, values file, and `--set` flags as your actual `helm install`/`helm upgrade`. If they differ, the rendered manifest will not match the running deployment (wrong labels, secret name, or ports).

> [!NOTE]
> In your actual values file, keep the chart's `PodMonitor` disabled — `prometheusMonitoring.podMonitor.enabled: false` for non-DCE editions, `applicationNodes.prometheusMonitoring.podMonitor.enabled: false` for Data Center Edition. The native `PodMonitor` targets the standard Prometheus Operator CRD, which Azure managed Prometheus ignores, so there is no reason to deploy it. The flag is set to `true` in the `helm template` commands below only to force the template to render so it can be extracted and converted.

The chart reference differs by edition: non-DCE editions use `sonarqube/sonarqube`, while Data Center Edition uses `sonarqube/sonarqube-dce` (and enables the exporter via `applicationNodes.prometheusExporter.enabled`).

#### Community, Developer, and Enterprise editions

Bash:

```bash
helm template sonarqube sonarqube/sonarqube \
  -n sonarqube \
  -f ./values.yaml \
  --set prometheusMonitoring.podMonitor.enabled=true \
  --set prometheusExporter.enabled=true \
  --show-only templates/prometheus-podmonitor.yaml \
  | sed 's#monitoring.coreos.com/v1#azmonitoring.coreos.com/v1#' > sonarqube-podmonitor.yaml
```

PowerShell:

```powershell
$(helm template sonarqube sonarqube/sonarqube `
  -n sonarqube `
  -f .\values.yaml `
  --set prometheusMonitoring.podMonitor.enabled=true `
  --set prometheusExporter.enabled=true `
  --show-only templates/prometheus-podmonitor.yaml) -replace "monitoring.coreos.com/v1", "azmonitoring.coreos.com/v1" > sonarqube-podmonitor.yaml
```

#### Data Center Edition

Bash:

```bash
helm template sonarqube sonarqube/sonarqube-dce \
  -n sonarqube \
  -f ./values.yaml \
  --set applicationNodes.prometheusMonitoring.podMonitor.enabled=true \
  --set applicationNodes.prometheusExporter.enabled=true \
  --show-only templates/prometheus-podmonitor.yaml \
  | sed 's#monitoring.coreos.com/v1#azmonitoring.coreos.com/v1#' > sonarqube-podmonitor.yaml
```

PowerShell:

```powershell
$(helm template sonarqube sonarqube/sonarqube-dce `
  -n sonarqube `
  -f .\values.yaml `
  --set applicationNodes.prometheusMonitoring.podMonitor.enabled=true `
  --set applicationNodes.prometheusExporter.enabled=true `
  --show-only templates/prometheus-podmonitor.yaml) -replace "monitoring.coreos.com/v1", "azmonitoring.coreos.com/v1" > sonarqube-podmonitor.yaml
```

### Static manifest examples

If you prefer to write the manifest by hand, adapt one of the following to your deployment.

> [!NOTE]
> Two values usually need adjusting: the `namespace` (and the matching `namespaceSelector`) where SonarQube runs, and the `bearerTokenSecret.name` of the monitoring-passcode secret. See [Referencing the system passcode](#referencing-the-system-passcode) for how to find the secret name.

#### Community, Developer, and Enterprise editions

```yaml
apiVersion: azmonitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: sonarqube
  namespace: sonarqube                                # Must be the namespace where SonarQube is deployed
  labels:
    app: sonarqube
spec:
  namespaceSelector:
    matchNames:
    - sonarqube
  selector:
    matchLabels:
      app: sonarqube
  podMetricsEndpoints:
  - port: http
    path: /api/monitoring/metrics
    scheme: http
    interval: 30s
    bearerTokenSecret:
      name: sonarqube-sonarqube-monitoring-passcode   # Must be the name of the secret containing the monitoring passcode
      key: SONAR_WEB_SYSTEMPASSCODE
  - port: monitoring-ce
    path: /
    scheme: http
    interval: 30s
  - port: monitoring-web
    path: /
    scheme: http
    interval: 30s
```

#### Data Center Edition

```yaml
apiVersion: azmonitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: sonarqube
  namespace: sonarqube                                      # Must be the namespace where SonarQube is deployed
  labels:
    app: sonarqube-dce
spec:
  namespaceSelector:
    matchNames:
    - sonarqube
  selector:
    matchLabels:
      app: sonarqube-dce
  podMetricsEndpoints:
  - port: http
    path: /api/monitoring/metrics
    scheme: http
    interval: 30s
    bearerTokenSecret:
      name: sonarqube-dce-sonarqube-dce-monitoring-passcode  # Must be the name of the secret containing the monitoring passcode
      key: SONAR_WEB_SYSTEMPASSCODE
  - port: monitoring-ce
    path: /
    scheme: http
    interval: 30s
  - port: monitoring-web
    path: /
    scheme: http
    interval: 30s
```

> [!NOTE]
> The `apiVersion: azmonitoring.coreos.com/v1` group is used by the Azure Monitor metrics addon. Do not use `monitoring.coreos.com/v1` (the standard Prometheus Operator group) — that version is not processed by the Azure Monitor agent.

## Referencing the system passcode

> [!NOTE]
> If you generated the manifest with `helm template` above, the `bearerTokenSecret` wiring is already correct — you can skip to [Applying the manifest](#applying-the-manifest). This section applies when adapting a static example by hand.

The `PodMonitor` authenticates to the `/api/monitoring/metrics` endpoint with the system passcode via `bearerTokenSecret`. The `name` and `key` you set there depend on how the passcode is managed:

- **Chart-managed passcode** — if you let the Helm chart generate the passcode, it stores it in a secret for you. The name is derived from your Helm release name, so it differs between the non-DCE and DCE charts and with your chosen release name (hence the `sonarqube-sonarqube-...` and `sonarqube-dce-sonarqube-dce-...` examples above). Find the actual name in your cluster:

  ```bash
  # Returns just the secret names
  kubectl get secrets -A -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}' | grep monitoring

  # Or, more simply
  kubectl get secrets -A | grep monitoring
  ```

- **Externally-managed passcode** — if you supply the passcode through your own secret (configured in the Helm values via `monitoringPasscodeSecretName` and `monitoringPasscodeSecretKey`), set `bearerTokenSecret.name` and `bearerTokenSecret.key` to those same values.

Either way, make sure the referenced secret lives in the namespace selected by the `PodMonitor`.

## Applying the manifest

Apply the manifest — the file generated above (`sonarqube-podmonitor.yaml`), or the static example you saved to a file:

```bash
kubectl apply -f sonarqube-podmonitor.yaml
```

## Verifying scraping in Azure Monitor

After applying the manifest, allow a few minutes for the Azure Monitor agent to pick up the new configuration.

### Check targets in the Azure portal

1. Open your Azure Monitor workspace in the Azure portal
2. Navigate to **Prometheus explorer** or use **Metrics** with the Prometheus data source
3. Run a basic query (e.g. `up`) and filter by the SonarQube pods to confirm metrics are arriving

Look for results with labels matching the SonarQube pods — expect one series per scraped pod, so a multi-pod deployment returns several. To confirm the SonarQube Prometheus endpoint specifically, query `sonarqube_web_uptime_minutes`.

## Next steps

- Set up the [Grafana dashboard](../../dashboards/sonarqube-grafana-prometheus-k8s/) using Azure Monitor as a Prometheus data source
- Review the [Prometheus queries](../prometheus-queries.md) reference for recommended alert conditions
- To configure alerts, use Azure Monitor alert rules in the Azure portal against your managed Prometheus workspace
