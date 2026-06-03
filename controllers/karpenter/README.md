# controllers/karpenter

Local Helm chart that delivers Karpenter's `EC2NodeClass` and `NodePool` definitions to every cluster. Rendered by `../../apps/karpenter-resources.yaml` (`ApplicationSet`, sync wave 1) once the upstream Karpenter chart at `../../apps/karpenter.yaml` (sync wave 0) has installed the CRDs.

## Why a chart and not raw YAML

ArgoCD's `directory` source applies YAML as-is — there is no value substitution into manifest bodies. The `ApplicationSet` cluster generator can only substitute into the `Application` spec, not into the manifests the `Application` references. So either:

1. The manifests are hardcoded per cluster (account-identifying values committed publicly — not acceptable), or
2. The manifests are templated.

Helm wins over kustomize here because the substitution surface is small (three values, listed below) and `ApplicationSet`'s `helm.valuesObject` is the standard injection point.

## Values

| Key | Source | Purpose |
|---|---|---|
| `clusterName` | ArgoCD cluster `Secret` (`{{ .values.clusterName }}`) | Value of the `karpenter.sh/discovery` tag — Karpenter uses it for subnet, security-group, and launched-instance tag selection. Must match `local.cluster_name` in the infrastructure repo's EKS module. |
| `nodeRoleName` | ArgoCD cluster `Secret` (`{{ .values.karpenterNodeRoleName }}`) | IAM role NAME (not ARN) assumed by Karpenter-launched instances. In Karpenter v1, `EC2NodeClass.spec.role` is a role name; Karpenter manages the instance profile itself. |
| `nodePoolLimits.cpu` | Chart default (`200`), overridable per cluster | Cluster-wide vCPU cap per pool — runaway cost guardrail. |
| `nodePoolLimits.memory` | Chart default (`200Gi`), overridable per cluster | Cluster-wide memory cap per pool. |

## Resources

- `templates/ec2nodeclass.yaml` — single `EC2NodeClass` named `default`, Bottlerocket via the `@latest` alias, tag-based subnet/SG selection.
- `templates/nodepool-general-purpose.yaml` — arm64 (Graviton) NodePool, spot-first, c8g/m8g/r8g + c7g/m7g/r7g, sizes medium..8xlarge, 30-day expiry, weight 100.
- `templates/nodepool-general-purpose-amd64.yaml` — amd64 fallback NodePool with identical shape on Intel/AMD silicon, weight 10.

## Extending

To add a specialised NodePool (GPU, memory-intensive, on-demand-only):

1. Add a new `templates/nodepool-<name>.yaml` with the appropriate `requirements` and a `taints:` block so only opted-in pods schedule there.
2. If the new pool needs a different AMI family or a different instance profile, add a sibling `EC2NodeClass` and reference it from the new NodePool's `nodeClassRef`.
3. Open a PR. No infrastructure change is required — Karpenter picks up the new pool immediately on next reconcile.
