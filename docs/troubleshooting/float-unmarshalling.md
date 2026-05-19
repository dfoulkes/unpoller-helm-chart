# Float Unmarshalling Bug (v2.1.3)

## Problem

UnPoller v2.1.3 (image `golift/unifi-poller:2.1.3`) has a JSON unmarshalling bug where it expects `int64` for the `tx_bytes-r` field, but the UniFi Controller API returns floating-point values.

### Symptoms

Pod logs show continuous errors:

```
2026/05/19 16:34:02 [ERROR] metric fetch for InfluxDB failed: unifi.GetClients(https://192.168.50.1): 
json: cannot unmarshal number 0.0 into Go struct field Client.data.tx_bytes-r of type int64
```

**Result:** No UnPoller client metrics are exported. Queries for `unpoller_client_*` return empty in Prometheus/Grafana.

### Root Cause

The UniFi Controller API sometimes returns float values like:
- `0.0`
- `320.0362647325476`
- `46.59498207885304`

UnPoller v2.1.3's Go unmarshaller expects these as `int64`, causing a type mismatch and rejecting the entire client metrics payload.

## Solution

Upgrade to UnPoller v2.11.2 — this version handles float values gracefully.

### Chart Update

Update `Chart.yaml`:

```yaml
version: "2.2.0"
appVersion: "v2.11.2"
```

The chart already uses the correct image repository (`ghcr.io/unpoller/unpoller`), so updating `appVersion` will pull v2.11.2 automatically.

### Verification

After upgrading:

1. **Check pod logs** — should show successful scrapes:
   ```bash
   kubectl logs -n monitoring -l app.kubernetes.io/name=unpoller --tail=50
   ```
   
   Expected output:
   ```
   2026/05/19 16:51:04 [INFO] Found 1 site(s) on controller https://192.168.50.1: default (Default)
   2026/05/19 16:51:04 [INFO] Prometheus exported at http://0.0.0.0:9130/ - namespace: unpoller
   ```

2. **Verify metrics endpoint**:
   ```bash
   kubectl port-forward -n monitoring svc/unpoller 9130:9130
   curl http://localhost:9130/metrics | grep unpoller_client_roam_count_total
   ```
   
   Should return data for all connected clients.

3. **Check Prometheus**:
   ```promql
   unpoller_client_roam_count_total
   ```
   
   Should return ~30+ time series (one per client).

## Prevention

- **Pin versions** — don't use `latest` tag; specify exact versions in `appVersion`
- **Monitor metrics flow** — add alerts for `rate(unpoller_client_roam_count_total[5m]) == 0`

## Timeline

- **2026-05-18 06:33** — UnPoller v2.1.3 deployed (incorrect version)
- **2026-05-19 16:44** — Issue discovered (no client metrics)
- **2026-05-19 16:49** — Upgraded to v2.11.2, metrics restored

## References

- [GitHub Issue](https://github.com/unpoller/unpoller/issues) - Similar float parsing issues
- [UnPoller Changelog](https://github.com/unpoller/unpoller/releases) - v2.7+ includes float handling fixes
