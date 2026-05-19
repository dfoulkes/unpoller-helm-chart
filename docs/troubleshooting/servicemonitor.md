# ServiceMonitor Configuration

## Problem

UnPoller pod is running and exporting metrics at `:9130/metrics`, but Prometheus isn't scraping it — `unpoller_client_*` queries return empty.

### Symptoms

1. **Metrics endpoint works**:
   ```bash
   kubectl port-forward -n monitoring svc/unpoller 9130:9130
   curl http://localhost:9130/metrics | grep unpoller_client
   # Returns data ✓
   ```

2. **Prometheus has no target**:
   ```bash
   curl http://192.168.55.12:9090/api/v1/targets | grep unpoller
   # Empty ✗
   ```

3. **PodMonitor exists but not discovered**:
   ```bash
   kubectl get podmonitor -n monitoring
   # unifi-poller exists ✓
   ```

### Root Cause

The Prometheus Operator's `serviceMonitorSelector` is configured to only watch resources with specific labels:

```yaml
serviceMonitorSelector:
  matchLabels:
    release: prometheus
```

The PodMonitor created by the Helm chart doesn't have this label, so Prometheus never discovers it.

## Solution

Create a ServiceMonitor with the correct label in your infrastructure repo (e.g., `prometheus-setup`):

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: unpoller
  namespace: monitoring
  labels:
    release: prometheus  # ← Required for discovery
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: unpoller
  endpoints:
    - port: tcp
      interval: 30s
      scrapeTimeout: 10s
      path: /metrics
      relabelings:
        - sourceLabels: [__meta_kubernetes_pod_name]
          targetLabel: instance
        - sourceLabels: [__meta_kubernetes_namespace]
          targetLabel: region
        - replacement: unpoller
          targetLabel: service
        - replacement: unpoller
          targetLabel: job
```

### Why ServiceMonitor instead of PodMonitor?

While the Helm chart includes a PodMonitor, using a ServiceMonitor in your infrastructure repo:

1. **Follows established patterns** — other exporters use ServiceMonitors
2. **Centralizes monitoring config** — scrape config lives in `prometheus-setup`, not scattered across app Helm charts
3. **Allows custom relabeling** — standardize labels across all scraped services

## Verification

1. **Apply the ServiceMonitor**:
   ```bash
   kubectl apply -f monitoring/unpoller/unpoller-servicemonitor.yaml
   ```

2. **Wait 30s for Prometheus to discover it**

3. **Check targets**:
   ```bash
   curl -s "http://192.168.55.12:9090/api/v1/targets" | grep unpoller
   ```
   
   Should show:
   ```json
   {
     "labels": {
       "job": "monitoring/unifi-poller"
     },
     "health": "up"
   }
   ```

4. **Query metrics**:
   ```promql
   unpoller_client_roam_count_total
   ```
   
   Should return data within 1-2 minutes.

## Best Practice

**Don't rely on the chart's PodMonitor alone** — always create a ServiceMonitor in your infrastructure repo with the correct `release: prometheus` label to ensure Prometheus discovery works.

## Timeline

- **2026-05-19 16:49** — UnPoller deployed with PodMonitor (not discovered)
- **2026-05-19 17:00** — ServiceMonitor added to prometheus-setup repo
- **2026-05-19 17:02** — Prometheus discovered target, metrics flowing

## References

- [Prometheus Operator ServiceMonitor](https://github.com/prometheus-operator/prometheus-operator/blob/main/Documentation/user-guides/getting-started.md#related-resources)
- [prometheus-setup examples](https://github.com/dfoulkes/prometheus-setup/tree/main/monitoring)
