# network-policies

Baseline **default-deny** `CiliumNetworkPolicy` set for application namespaces,
reconciled by ArgoCD (`apps/network-policies.yaml`).

## What it ships

For every namespace in `appNamespaces`:

| Policy | Effect |
|---|---|
| `default-deny` | Selects all pods with empty `ingress: []` / `egress: []` rules — the Cilium deny-all idiom (empty rules array = default-deny with zero allowed peers). The zero-trust floor. |
| `allow-dns` | Allows egress to `kube-dns` on port 53, via Cilium's L7 DNS proxy (queried names show up in Hubble). Every workload needs this. |
| `allow-intra-namespace` | *Optional* (`allowIntraNamespace`, default off). Opens pod-to-pod traffic within the same namespace. |

Everything else — cross-namespace traffic, egress to the internet, app → Aurora,
app → Redis, ingress → app — stays denied until an explicit allow policy is added.
Those per-workload allows are versioned **with the workloads they describe** (the
app's own chart), not here, so least-privilege networking tracks the app it
protects.

## Why these are GitOps, not Terraform

The Cilium control plane (agent, operator, Hubble) is a bootstrap primitive and
installs via Terraform (`modules/cilium` in the infra repo) — a cluster can't run
a pod, including ArgoCD, without a CNI. The **policies** that ride on top iterate
at application cadence and are exactly the declarative in-cluster state ArgoCD
should own. See the infra journal entry "Cilium install boundary".

## Values

| Key | Default | Purpose |
|---|---|---|
| `appNamespaces` | `[demo]` | Namespaces that receive the baseline. |
| `createNamespaces` | `true` | Create the Namespace objects. Set false where the app's chart already creates them. |
| `allowIntraNamespace` | `false` | Render the optional intra-namespace allow. |

The `demo` namespace is a validation fixture: it lets the default-deny behaviour
be proven (two pods, traffic dropped until an allow is added, drops visible in
Hubble) without waiting for the application to exist.

## ci/values.yaml

CI render stub so `helm template`/`helm lint` can validate the chart's schema in
the manifests pipeline, independent of runtime values.
