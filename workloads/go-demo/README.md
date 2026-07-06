# go-demo

The [Go demo service](https://github.com/RamiroCuenca/eks-platform-demo-app), one image in two roles: an HTTP server exercising Aurora (TLS enforced) and Redis (TLS + AUTH), and a queue worker draining the Redis list the server's `/enqueue` fills. It runs in the default-deny `demo` namespace with a workload-owned FQDN egress policy.

## Sync choreography (waves inside this app)

| Wave | Resources | Why |
|---|---|---|
| 0 | ServiceAccounts, SecretProviderClasses, network policy, Service | Identities and mounts must exist before anything consumes them |
| 1 | `db-init` Job (Sync hook, `BeforeHookCreation`) | Provisions the least-privilege DB user as the only identity that can read the master secret; idempotent SQL makes per-sync re-runs harmless |
| 2 | server + worker Deployments, HPA, scrape configs | Deploy only after the DB user exists — `/readyz` gates traffic on live Aurora + Redis connectivity |

## Secret vs fact — how configuration arrives

- **Secret material** (app-user DB password, Redis AUTH token): CSI tmpfs file mounts, extracted from their JSON secrets by `jmesPath`. The app reads `*_FILE` paths; values never touch the process environment.
- **Non-secret facts** (endpoints, ports, dbname, DB username, image repository): injected by `apps/go-demo.yaml` from the ArgoCD cluster Secret's `platform.io/*` annotations into plain env. Nothing account-specific or apply-generated is committed here.
- **The image tag** is the one deployment-defining committed value. The app pipeline promotes it by auto-merged pull request after each publish; ECR tag immutability guarantees the tag names the exact artifact that passed the gates.

## Identity split

`go-demo` (server + worker) can read only its own credential and the Redis connection secret. `go-demo-db-init` (the Job) is the only identity able to read the RDS master secret. The app user itself holds `CONNECT` and nothing else — the service performs no relational reads or writes.

## Observability

Scrape configs are workload-owned, like the network policies: a
`ServiceMonitor` for the server (via its Service) and a `PodMonitor` for the
worker, which deliberately has no Service — it exposes `:8080` only for
`/healthz` and `/metrics`. Because the namespace is default-deny in both
directions, the chart also ships `allow-monitoring-scrape`, admitting
Prometheus pods from the `monitoring` namespace on the metrics port; without
it, scrapes fail *silently* (targets appear, every series is just absent).

## Scaling

CPU-based HPA on the server (the Deployment omits `spec.replicas` so ArgoCD self-heal and the autoscaler never contend). The worker is fixed at one replica until the autoscaling phase hands it to KEDA's Redis-list scaler.
