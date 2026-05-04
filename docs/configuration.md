# Configuration

This document covers Radar's cluster connection behavior. For CLI flags and basic usage, see the [README](../README.md#usage).

## Persistent Configuration

Radar stores configuration in two files under `~/.radar/`:

### Config File (`~/.radar/config.json`)

Persistent defaults for CLI flags. CLI flags always override these values. Managed via the Settings dialog in the UI or `PUT /api/config`.

```json
{
  "kubeconfig": "",
  "kubeconfigDirs": [],
  "namespace": "",
  "port": 9280,
  "noBrowser": false,
  "timelineStorage": "memory",
  "timelineDbPath": "~/.radar/timeline.db",
  "historyLimit": 10000,
  "prometheusUrl": "",
  "mcp": true
}
```

All fields are optional — omitted fields use built-in defaults.

| Field | Description |
|-------|-------------|
| `kubeconfig` | Path to kubeconfig file (same as `--kubeconfig`) |
| `kubeconfigDirs` | Directories containing kubeconfig files (same as `--kubeconfig-dir`) |
| `namespace` | Initial namespace filter |
| `port` | Server port (default 9280) |
| `noBrowser` | Don't auto-open browser |
| `timelineStorage` | `memory` or `sqlite` |
| `timelineDbPath` | Path to SQLite database |
| `historyLimit` | Max timeline events to retain |
| `prometheusUrl` | Manual Prometheus/VictoriaMetrics URL — skips auto-discovery. Useful when Prometheus is not in the same cluster or uses a non-standard service name. |
| `mcp` | Enable/disable MCP server for AI tools (default: enabled) |

### Settings File (`~/.radar/settings.json`)

User preferences for the UI. Managed via the Settings dialog or `PUT /api/settings`.

```json
{
  "theme": "system",
  "pinnedKinds": [
    { "name": "Deployments", "kind": "Deployment", "group": "" }
  ]
}
```

| Field | Values | Description |
|-------|--------|-------------|
| `theme` | `light`, `dark`, `system` | UI theme preference |
| `pinnedKinds` | Array of `{name, kind, group}` | Resource kinds pinned to the sidebar |

## Cluster Connection Precedence

Radar connects to Kubernetes clusters using the same configuration sources as `kubectl`:

| Priority | Source | Description |
|----------|--------|-------------|
| 1 | `--kubeconfig` flag | Explicit path to kubeconfig file |
| 2 | `KUBECONFIG` env var / `--kubeconfig-dir` flag | Either can provide kubeconfig(s); mutually exclusive alternatives |
| 3 | In-cluster config | Automatic when running inside a Kubernetes pod (`KUBERNETES_SERVICE_HOST` is set) |
| 4 | `~/.kube/config` | Default kubeconfig location |

## KUBECONFIG vs In-Cluster Detection

When Radar runs inside a Kubernetes pod, Kubernetes automatically sets the `KUBERNETES_SERVICE_HOST` environment variable. This normally triggers in-cluster configuration using the pod's service account credentials.

However, **explicit kubeconfig takes precedence**. If you set `KUBECONFIG` or pass `--kubeconfig`, Radar uses that instead of in-cluster config. This allows you to:

- Run Radar inside a pod but connect to a different cluster
- Use specific credentials instead of the pod's service account
- Test with a custom kubeconfig while developing inside a cluster

**Example: Override in-cluster config**
```bash
# Inside a pod, connect to a different cluster
export KUBECONFIG=/path/to/other-cluster.yaml
kubectl radar
```

This behavior matches `kubectl` and follows the [Kubernetes client-go precedence rules](https://github.com/kubernetes/kubernetes/issues/43662).

## Multiple Kubeconfig Files

`KUBECONFIG` can contain multiple file paths (colon-separated on Linux/macOS, semicolon-separated on Windows). Radar merges these files following Kubernetes conventions:

```bash
export KUBECONFIG=~/.kube/config:~/.kube/staging-config:~/.kube/prod-config
kubectl radar
```

Alternatively, use `--kubeconfig-dir` to load all kubeconfig files from a directory:

```bash
kubectl radar --kubeconfig-dir ~/.kube/configs/
```

## Context Switching

Radar supports switching between Kubernetes contexts at runtime through the UI. Click the context selector in the header to switch between available contexts.

When running in-cluster (using the pod's service account), context switching is disabled.

## Metrics (Prometheus / VictoriaMetrics)

Radar connects to a Prometheus-compatible metrics endpoint to power the workload CPU/memory charts and the Cost Insights view. It uses a four-layer discovery strategy:

1. **Manual URL override** — if `--prometheus-url` (or `prometheusUrl` in `~/.radar/config.json`) is set, Radar uses that URL and skips all discovery.
2. **Existing port-forward** — if a traffic-system port-forward is already active for the current context, Radar reuses it.
3. **Well-known service names** — Radar checks a list of common Prometheus/VictoriaMetrics service names across standard monitoring namespaces (`monitoring`, `victoria-metrics`, `prometheus`, `observability`, `metrics`, `kube-system`, `default`, `opencost`, `caretta`). If a matching service is found and reachable from inside the cluster, it connects directly. If it's not reachable (e.g. you're running Radar locally), Radar starts a `kubectl port-forward` automatically.
4. **Dynamic cluster-wide discovery** — if no well-known services match, Radar scores all cluster services by labels, port numbers, and name patterns, then connects to or port-forwards to the best candidate.

### Running locally against a remote cluster

When Radar runs on your laptop, cluster-internal addresses (`*.svc.cluster.local`) are not reachable. For well-known services Radar will attempt an automatic `kubectl port-forward` on your behalf. If that fails (RBAC restrictions, non-standard service name, etc.), pass the URL directly via `--prometheus-url`.

**Option 1 — let Radar port-forward automatically**

If your metrics service name and namespace match one of the well-known locations (see list above), Radar handles the port-forward for you. No extra steps needed.

**Option 2 — manual `kubectl port-forward`**

Port-forward your metrics service in one terminal, then start Radar with `--prometheus-url` pointing at the local port:

```bash
# VictoriaMetrics (single-node, default port 8428)
kubectl port-forward -n monitoring svc/victoria-metrics-single-server 8428:8428

# Prometheus (default port 9090)
kubectl port-forward -n monitoring svc/prometheus-operated 9090:9090

# In another terminal:
kubectl radar --prometheus-url http://localhost:8428
# or for Prometheus:
kubectl radar --prometheus-url http://localhost:9090
```

For VictoriaMetrics Cluster (vmselect), include the API sub-path:

```bash
kubectl port-forward -n monitoring svc/vmselect 8481:8481
kubectl radar --prometheus-url http://localhost:8481/select/0/prometheus
```

**Option 3 — persist via config file**

Add `prometheusUrl` to `~/.radar/config.json` so you don't have to pass the flag every time:

```json
{
  "prometheusUrl": "http://localhost:8428"
}
```

## Related Documentation

- [README](../README.md#usage) — CLI flags and basic usage
- [In-Cluster Deployment](in-cluster.md) — Deploy Radar inside your cluster with Helm
- [Authentication & Authorization](authentication.md) — Proxy and OIDC auth for shared deployments
