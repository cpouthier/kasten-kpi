# Deployment methodology — new Prometheus instance + Kasten remote_write

Step-by-step runbook to stand up a receiving Prometheus, wire Veeam
Kasten's `remote_write` into it, and get `kpi-dashboard.json` showing real
data. This is the exact sequence used in production (see the `homelab`
reference implementation's `deploy.sh`), broken out step by step so it can
be adapted to any cluster.

See [README.md](README.md) for the underlying *why* behind each
prerequisite — this file is the *how*, in order.

---

## Step 1 — Deploy the receiving Prometheus

This Prometheus scrapes nothing on its own — it only **receives** what
Kasten pushes to it, plus one targeted scrape of kube-state-metrics for
namespace discovery.

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts --force-update
helm repo update prometheus-community
```

`values-prometheus.yaml`:
```yaml
server:
  extraFlags:
    - web.enable-lifecycle
    - web.enable-remote-write-receiver   # required — without it, remote_write is rejected
  persistentVolume:
    size: 10Gi
alertmanager:
  enabled: false
prometheus-node-exporter:
  enabled: false
prometheus-pushgateway:
  enabled: false

# Needed so $namespace and the cluster-wide compliance panel know which
# namespaces genuinely exist (see step 4)
kube-state-metrics:
  enabled: true

# Every default scrapeConfig in the chart MUST be disabled — otherwise this
# Prometheus scrapes the whole cluster in addition to receiving remote_write.
scrapeConfigs:
  prometheus: {enabled: false}
  kubernetes-api-servers: {enabled: false}
  kubernetes-nodes: {enabled: false}
  kubernetes-nodes-cadvisor: {enabled: false}
  kubernetes-service-endpoints: {enabled: false}
  kubernetes-service-endpoints-slow: {enabled: false}
  prometheus-pushgateway: {enabled: false}
  kubernetes-services: {enabled: false}
  kubernetes-pods: {enabled: false}
  kubernetes-pods-slow: {enabled: false}
  # Only active scrape: targeted, namespace discovery only
  kube-state-metrics:
    enabled: true
    job_name: kube-state-metrics
    static_configs:
      - targets: ["<release>-kube-state-metrics.<namespace>.svc.cluster.local:8080"]
    metric_relabel_configs:
      - action: keep
        source_labels: [__name__]
        regex: kube_namespace_created|kube_namespace_status_phase
```

```bash
helm upgrade --install prometheus prometheus-community/prometheus \
  --namespace monitoring --create-namespace \
  -f values-prometheus.yaml --wait --timeout 5m
```

---

## Step 2 — Deploy Grafana

```bash
helm repo add grafana https://grafana.github.io/helm-charts --force-update
```

`values-grafana.yaml`:
```yaml
adminPassword: "<password>"
service:
  type: LoadBalancer   # or ClusterIP + Ingress, depending on your setup
persistence:
  enabled: true
  size: 2Gi
# Avoids a deadlock between an RWO PVC and the default RollingUpdate
# strategy (the new pod can't mount a volume still attached to the old
# one) — hit in practice.
deploymentStrategy:
  type: Recreate
datasources:
  datasources.yaml:
    apiVersion: 1
    datasources:
      - name: Prometheus
        uid: prometheus        # must exactly match kpi-dashboard.json
        type: prometheus
        url: http://prometheus-server.monitoring.svc.cluster.local
        access: proxy
        isDefault: true
dashboardProviders:
  dashboardproviders.yaml:
    apiVersion: 1
    providers:
      - name: default
        orgId: 1
        folder: ""
        type: file
        disableDeletion: false
        editable: true
        options:
          path: /var/lib/grafana/dashboards/default
```

```bash
helm upgrade --install grafana grafana/grafana \
  --set-file "dashboards.default.veeam-kasten-kpi.json=./kpi-dashboard.json" \
  --namespace monitoring --create-namespace \
  -f values-grafana.yaml --wait --timeout 5m
```

---

## Step 3 — Enable remote_write on the Kasten side

This is the step that breaks most often: the default keep-regex from
[Observability Series 1](https://veeamkasten.dev/observability-series-1-prometheus-remote-writes)
does **not** include `catalog_actions_count`, the metric the whole KPI
dashboard depends on.

```yaml
# values-k10-observability.yaml
clusterName: <cluster-name>   # required once remote_write is enabled
prometheus:
  server:
    remote_write:
      - url: http://prometheus-server.monitoring.svc.cluster.local/api/v1/write
        write_relabel_configs:
          - action: keep
            source_labels: [__name__]
            regex: |
              action_.*|.*_persistent_volume_.*|repository_data_.*|data_operation_.*|data_upload_session_.*|exec_.*|limiter_.*|export_storage_.*|snapshot_storage_.*|metering_license_.*|events_service_.*|process_.*|compliance_count|catalog_storage_artifact_count|catalog_actions_count|retire_lag_seconds
```

```bash
helm upgrade k10 kasten/k10 -n kasten-io \
  --reuse-values -f values-k10-observability.yaml --wait --timeout 3m
```

---

## Step 4 — Verify data is actually arriving in Prometheus, before touching Grafana

```bash
# The dashboard's key metric — must exist with namespace/policy/status/type labels
curl -s "http://<prometheus-host>/api/v1/query?query=catalog_actions_count" | jq .

# kube-state-metrics must expose the real namespaces
curl -s "http://<prometheus-host>/api/v1/query?query=kube_namespace_created" | jq .
```

If either returns `"result": []`, don't move on to the dashboard yet — the
problem is upstream (the remote_write keep-regex for the first one, the
scrape config for the second).

---

## Step 5 — Import the dashboard

```bash
curl -s -u admin:<password> http://<grafana-host>/api/datasources | jq '.[] | select(.type=="prometheus") | .uid'
```

- If the returned uid is `prometheus` → `kpi-dashboard.json` imports as-is.
- Otherwise → replace all 6 occurrences of `"uid": "prometheus"` in the JSON
  with that uid before importing (see [README.md](README.md) for exact
  line numbers and a `grep`/`sed` recipe).

---

## Step 6 — Final check inside Grafana

1. Open "Veeam Kasten KPI" — the `$namespace` dropdown should list your real
   namespaces (not empty).
2. Set `$rpo_target` to a realistic value, in hours.
3. The "Application RPO Status" table should show every application with a
   sane "Time since last success" — this is the most reliable signal that
   the whole chain works, before trusting the percentage panels.
