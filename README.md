# rkn-block-checker (Helm chart)

Prometheus exporter that periodically probes a list of URLs and reports
whether each one is in an RKN/TSPU-blocked zone, broken down by the
layer that failed (DNS / TCP / TLS / HTTP).

The exporter wraps the existing `rkn_checker` library, so verdicts and
confidence levels match the CLI 1:1.

## Quick start

```bash
helm install rkn ./charts/rkn-block-checker -f values-example.yaml
```

Minimal config:

```yaml
targets:
  - name: cdn-prod
    url: https://cdn-prod.thehood.game
    expect_available: true

serviceMonitor:
  enabled: true
  labels:
    release: prometheus     # kube-prometheus-stack scrape selector

prometheusRule:
  enabled: true
  labels:
    release: prometheus

dashboard:
  enabled: true
  namespace: monitoring     # Grafana sidecar namespace
```

## What gets deployed

| Resource | Purpose |
|---|---|
| `Deployment` | the exporter, one container, port 9090 |
| `Service` | ClusterIP exposing `/metrics`, `/healthz`, `/readyz` |
| `ConfigMap` (targets) | the URL list mounted at `/etc/rkn/targets.yaml` |
| `ConfigMap` (dashboard) | Grafana dashboard JSON, label `grafana_dashboard: "1"` |
| `ServiceAccount` | dedicated SA, no token automount |
| `ServiceMonitor` | Prometheus Operator scrape config |
| `PrometheusRule` | alerts: block detected, exporter down, no cycles |
| `PodDisruptionBudget` | opt-in, useful only with `replicaCount > 1` |
| `NetworkPolicy` | opt-in, restricts ingress to monitoring NS + egress to DNS/443/80 |

## Metrics surface

```
rkn_target_blocked{name,url,expect_available,verdict}   0|1
rkn_target_verdict_info{name,url,verdict,confidence}    1
rkn_target_dns_mismatch{name,url}                       0|1
rkn_target_tcp_ok{name,url}                             0|1
rkn_target_tls_ok{name,url}                             0|1
rkn_target_http_status{name,url}                        status code
rkn_target_tcp_seconds{name,url}                        seconds
rkn_target_tls_seconds{name,url}                        seconds
rkn_target_plt_seconds{name,url}                        seconds
rkn_target_last_check_timestamp_seconds{name,url}       epoch
rkn_check_duration_seconds_bucket{name,le}              histogram
rkn_check_errors_total{name,reason}                     counter
rkn_scrape_cycles_total                                 counter
rkn_build_info{version}                                 1
```

`reason` is one of `dns`, `tcp`, `tls`, `http`, `exception`.

`verdict` values come from `rkn_checker.models.Verdict`:
`OK, DNS_BLOCK, TCP_RESET, TLS_BLOCK, HTTP_STUB, TIMEOUT, DOWN, UNKNOWN`.

## Default alerts

| Alert | Fires when | Severity |
|---|---|---|
| `RKNBlockDetected` | `rkn_target_blocked{expect_available="true"} == 1` for 5m | critical |
| `RKNBlockResolved` | a previously blocked target returns to OK | info |
| `RKNExporterDown` | Prometheus loses the scrape target for 5m | critical |
| `RKNNoScrapeCycles` | no completed probe cycles in 15m | warning |
| `RKNBlockedLowConfidence` | blocked verdict with `confidence="LOW"` (opt-in) | warning |

All alert thresholds are configurable under `prometheusRule.rules.*`.

## Egress from Russia

The probes run wherever the pod runs. If the cluster is outside RU,
the metrics reflect "as seen from this cluster", which is not the RU
user view.

To probe from a Russian vantage point:

```yaml
checker:
  proxyUrl: socks5://ru-egress.internal:1080
```

The system DNS probe is intentionally **not** proxied — its job is to
detect ISP-level DNS poisoning, which means it has to go through the
RU resolver directly. Everything else (DoH, TCP, TLS, HTTP) traverses
the proxy. See the upstream README's "Privacy and threat model" for
the rationale.

## Replicas

The exporter is **idempotent** but not deduplicated: each replica probes
every target on its own schedule. Running `replicaCount: 2` doubles
outbound traffic from the cluster and produces two parallel series per
target (deduped by Prometheus on `instance`).

Recommended:

- `replicaCount: 1` — simplest, totally fine for monitoring
- `replicaCount: 2` + `podDisruptionBudget.enabled: true` — only if you
  need probe continuity during node drains; expect double egress

Sharding by target isn't built in; if you need it, run multiple releases
with non-overlapping `targets` lists.

## Image

Build locally:

```bash
docker build -t rkn-block-checker:dev .
```

Then `image.repository: rkn-block-checker, image.tag: dev`, `image.pullPolicy: IfNotPresent`.

## Verify

```bash
# templates render cleanly
helm template rkn ./charts/rkn-block-checker -f values-example.yaml | kubectl apply --dry-run=client -f -

# the exporter responds locally
kubectl port-forward svc/rkn 9090:9090
curl -s localhost:9090/metrics | grep rkn_target_blocked
```

## Uninstall

```bash
helm uninstall rkn
```

The dashboard ConfigMap survives if it was created in another namespace
(`dashboard.namespace`); delete it explicitly if needed.
