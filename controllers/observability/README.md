# observability

Platform scrape configs, alert rules and Grafana dashboards, reconciled by
ArgoCD (`apps/observability-config.yaml`, sync wave 1 — after
kube-prometheus-stack installs the `monitoring.coreos.com` CRDs at wave 0).

## What it ships

| Resource | Target | Why here |
|---|---|---|
| `ServiceMonitor/hubble` | `hubble-metrics` Service, kube-system :9965 | Flow metrics (dns, drop, tcp, flow, port-distribution, icmp) |
| `PodMonitor/cilium-agent` | agent pods :9962 | Datapath metrics — hostNetwork DaemonSet with no fronting Service, so a PodMonitor describes the topology honestly |
| `PodMonitor/cilium-operator` | operator pods :9963 | ENI/IPAM allocation health |
| `PrometheusRule/platform-alerts` | — | Node CPU saturation, demo-latency restart storm, go-demo SLO burn (5xx ratio, p95) |
| `ConfigMap/dashboard-*` | Grafana sidecar | `go-demo-slo` and `hubble-network` dashboards as code |

The Cilium **exporters** are enabled in the infra repo (Cilium installs at
cluster bootstrap, before the monitoring CRDs exist); the **scrape configs**
live here so a from-zero rebuild never races CRD installation. go-demo's own
monitors are not here either — workload-owned scrape configs live in the
workload's chart, same contract as its network policies.

## Alert rules

Additions on top of kube-prometheus-stack's `defaultRules` (CrashLoopBackOff,
node pressure, PVC fill and the rest of the fleet baseline ship there and are
not duplicated):

- `NodeCPUUtilizationHigh` — node CPU > 80% for 5m.
- `WorkloadPodRestartStorm` — >2 restarts in 5m in workload namespaces; the
  deliberate overlap with `KubePodCrashLooping`, which needs 15 quiet minutes
  a build-screenshot-destroy session never gives it.
- `GoDemoHighErrorRate` / `GoDemoP95LatencyHigh` — thresholds identical to
  the k6 load-test gates, so alerts and load tests agree on what "broken"
  and "too slow" mean.

Thresholds are values (`values.yaml`) so tuning is a one-line reviewable diff.

## Dashboards

Hand-written, focused JSONs — fleet-level views (nodes, namespaces,
workloads) already ship with kube-prometheus-stack's built-in dashboards and
are not duplicated. Cost-by-namespace was considered and cut: honest cost
allocation needs a cost model fed by CUR data (OpenCost/Kubecost), and
resource-requests-by-namespace is already covered by the built-ins.
