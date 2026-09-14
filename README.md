# kasten-kpi

A custom Grafana dashboard (`kpi-dashboard.json`) for **Veeam Kasten K10**
backup/export KPIs, built on top of Kasten's own Prometheus metrics — no
external agent, no extra scraping, just Kasten's `remote_write` feeding a
receiving Prometheus that Grafana reads from.

Panels:

- **Backup Success Rate** (`$policy` / `$namespace`) — share of backup
  attempts that completed successfully within the selected RPO window.
- **RPO Compliance (exported RP – cluster wide)** — % of applications
  currently on the cluster whose most recent successful *export* is within
  the RPO target.
- **RPO Compliance ($policy / $namespace)** — same freshness check, scoped
  to the current filter selection.
- **Application RPO Status** — per-application table: time since last
  successful export (`dd hh:mm:ss`) and an OK/BREACHED status against the
  RPO target.

All of it is driven by three template variables: `$policy`, `$namespace`,
and `$rpo_target` (in hours, editable live).

Built alongside, and complementary to, the official community dashboard —
[K10 Dashboard (21065)](https://grafana.com/grafana/dashboards/21065-k10-dashboard/)
— and based on the architecture described in:

- [Observability Series 1 — Prometheus remote writes](https://veeamkasten.dev/observability-series-1-prometheus-remote-writes)
- [Observability Series 2 — Grafana multi-cluster dashboard](https://veeamkasten.dev/observability-series-2-grafana-multi-cluster-dashboard)

---

## Prerequisites

This dashboard is **not** a drop-in import on a Grafana instance that merely
has 21065 already working — it depends on things that dashboard doesn't
need. Three things must be true on your cluster/Grafana before importing
`kpi-dashboard.json` as-is:

### 1. A Prometheus datasource with uid `prometheus`

Every panel and template variable hardcodes
`"datasource": {"type": "prometheus", "uid": "prometheus"}` — unlike 21065,
there's no templated `${DS_PROMETHEUS}` placeholder resolved on import. If
your Prometheus datasource has a different uid, every panel will show
*"Datasource prometheus was not found"*.

Fix: either provision your datasource with `uid: prometheus`, or
find-and-replace every `"uid": "prometheus"` in the JSON with your real
datasource uid before importing.

### 2. `catalog_actions_count` must be in Kasten's `remote_write` keep-regex

Every panel here — the `$policy` variable, Backup Success Rate, both RPO
Compliance panels, and the status table — is built on the single metric
`catalog_actions_count{namespace,policy,status,type,liveness}`.

This metric is **not** in the default keep-regex from
[Observability Series 1](https://veeamkasten.dev/observability-series-1-prometheus-remote-writes)
(`action_.*` does not match `catalog_actions_count`, since it doesn't start
with `action_`). If your Kasten `remote_write` config still uses that
blog's original regex unmodified, this dashboard will be permanently empty
even though 21065 works fine — 21065 relies on different metrics.

Fix: Kasten's Helm values need something like:

```yaml
clusterName: <your-cluster-name>
prometheus:
  server:
    remote_write:
      - url: http://<your-receiving-prometheus>/api/v1/write
        write_relabel_configs:
          - action: keep
            source_labels: [__name__]
            regex: |
              action_.*|.*_persistent_volume_.*|repository_data_.*|data_operation_.*|data_upload_session_.*|exec_.*|limiter_.*|export_storage_.*|snapshot_storage_.*|metering_license_.*|events_service_.*|process_.*|compliance_count|catalog_storage_artifact_count|catalog_actions_count|retire_lag_seconds
```

The important addition over the blog's original list is
`catalog_actions_count` (and `retire_lag_seconds` if you also want the
retire-lag metric).

### 3. `kube-state-metrics` exposing `kube_namespace_created`

The `$namespace` template variable is built from
`label_values(kube_namespace_created{...}, namespace)`, and the cluster-wide
RPO Compliance panel restricts itself to currently-live namespaces via
`and on(namespace) (kube_namespace_created)`. Without this metric on your
receiving Prometheus:

- The `$namespace` dropdown will be empty, and since it has no explicit
  `allValue`, "All" resolves to an empty regex — every panel filtered by
  `$namespace` will match nothing.
- The cluster-wide RPO Compliance panel will silently sit at a permanent
  **0%** (its `and on(namespace)` join returns nothing, and the panel's
  `or vector(0)` fallback masks this as "0% compliant" rather than "no
  data") — the most misleading failure mode of the three.

Fix: enable `kube-state-metrics` on your receiving Prometheus and scrape it
with a targeted job (not the cluster-wide default jobs), keeping only
`kube_namespace_created`/`kube_namespace_status_phase`:

```yaml
kube-state-metrics:
  enabled: true
scrapeConfigs:
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

### Not required, but worth knowing

This dashboard has **no `cluster_name` filter**, unlike 21065 (which needs
`${VAR_CLUSTER}`). If your receiving Prometheus gets `remote_write` from
more than one cluster at once (a genuine multi-cluster setup per
[Observability Series 2](https://veeamkasten.dev/observability-series-2-grafana-multi-cluster-dashboard)),
every panel here will silently aggregate across all of them — there's no
way to pick one cluster on this dashboard as it stands.

---

## Importing

Once the three prerequisites above hold:

```bash
# Option A — file-based Grafana provisioning:
helm upgrade --install grafana grafana/grafana \
  --set-file "dashboards.default.kasten-kpi.json=./kpi-dashboard.json" \
  -f values-grafana.yaml ...

# Option B — Grafana UI:
# Dashboards → New → Import → upload kpi-dashboard.json → pick your
# Prometheus datasource when prompted (only works if you removed the
# hardcoded "uid": "prometheus" occurrences first, or if your datasource
# already has that exact uid).
```

After import, open the dashboard, set `$rpo_target` to a realistic value in
hours, leave `$policy`/`$namespace` on "All", and confirm the "Application
RPO Status" table lists your real namespaces with sane "Time since last
success" values before trusting the percentage panels.
