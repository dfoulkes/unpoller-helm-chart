# UnPoller Helm Chart

Helm chart for deploying UnPoller — UniFi metrics exporter for Prometheus.

## Overview

UnPoller scrapes UniFi Dream Machine for network metrics and exposes them in Prometheus format. Tracks:

- Client roaming between APs (`unpoller_client_roam_count_total`)
- Device stats (APs, switches, gateway)
- Network performance metrics
- Traffic statistics

**Service endpoint:** `http://unpoller.monitoring.svc.cluster.local:9130/metrics`

## Installation

```bash
helm install unpoller ./charts/unpoller -n monitoring -f values-override.yaml
```

### Configuration

Create `values-override.yaml`:

```yaml
image:
  repository: "ghcr.io/unpoller/unpoller"
  pullPolicy: IfNotPresent

service:
  enabled: true
  type: ClusterIP
  port: 9130

podMonitor:
  enabled: true

upConfig: |
  [poller]
      debug = false
      quiet = false
      plugins = []
  [prometheus]
    disable = false
    http_listen = "0.0.0.0:9130"
    report_errors = false
  [influxdb]
    disable = true
  [unifi]
      dynamic = false
  [loki]
      disable = true
  [[unifi.controller]]    
      url         = "https://192.168.50.1"
      user        = "your-username"
      pass        = "your-password"
      sites       = ["all"]
      save_ids    = true
      save_dpi    = true
      save_sites  = true
      hash_pii    = false
      verify_ssl  = false
```

## Prometheus Integration

The chart includes a PodMonitor for Prometheus Operator, but you'll also need a ServiceMonitor with the correct label selector:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: unpoller
  namespace: monitoring
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: unpoller
  endpoints:
    - port: tcp
      interval: 30s
      scrapeTimeout: 10s
      path: /metrics
```

See [ServiceMonitor Configuration](troubleshooting/servicemonitor.md) for details.

## Troubleshooting

- [Float Unmarshalling Bug](troubleshooting/float-unmarshalling.md) - v2.1.3 parsing issue
- [ServiceMonitor Configuration](troubleshooting/servicemonitor.md) - Prometheus discovery setup

## Related

- [Prometheus Setup](https://github.com/dfoulkes/prometheus-setup)
- [UnPoller Documentation](https://unpoller.com)
