# eks-platform-gitops

[![sast](https://github.com/RamiroCuenca/eks-platform-gitops/actions/workflows/sast.yml/badge.svg?branch=main)](https://github.com/RamiroCuenca/eks-platform-gitops/actions/workflows/sast.yml)
[![secrets-scan](https://github.com/RamiroCuenca/eks-platform-gitops/actions/workflows/secrets-scan.yml/badge.svg?branch=main)](https://github.com/RamiroCuenca/eks-platform-gitops/actions/workflows/secrets-scan.yml)

GitOps configuration repository for the [eks-production-platform](https://github.com/RamiroCuenca/eks-production-platform). Kubernetes manifests, Helm chart references, and ArgoCD `ApplicationSet` definitions — reconciled into the cluster by ArgoCD.

---

## Relationship to the infrastructure repository

Two repositories cooperate to deliver the platform:

| Concern | Repository |
|---|---|
| AWS account scaffolding, VPC, IAM, EKS control plane, KMS | infra ([eks-production-platform](https://github.com/RamiroCuenca/eks-production-platform)) |
| Karpenter IRSA role, EC2 instance profile, SQS interruption queue | infra |
| ArgoCD install (Helm chart) + per-cluster `Secret` carrying cluster facts | infra |
| ArgoCD `ApplicationSet`s for every controller and workload | **this repo** |
| Karpenter Helm chart + `NodePool` / `EC2NodeClass` manifests | this repo |
| Cilium / `CiliumNetworkPolicy` | this repo |
| Observability stack (kube-prometheus-stack, Loki) | this repo |
| Demo workloads | this repo |

The split follows the standard rule of thumb: **AWS-credential-requiring resources go in the infrastructure repo; in-cluster declarative state lives here.** ArgoCD itself is the one bootstrap exception — the infrastructure repo installs it once and then steps out of the cluster.

## Repository layout

```
.
├── apps/                 # ArgoCD ApplicationSets, one per controller / workload
│   ├── karpenter.yaml              # upstream Karpenter chart, per cluster
│   └── karpenter-resources.yaml    # local NodePool/EC2NodeClass chart, per cluster
└── controllers/          # local Helm charts referenced by the ApplicationSets above
    └── karpenter/        # NodePool + EC2NodeClass for Karpenter
```

`apps/` is the only directory the in-cluster ArgoCD root Application points at. Adding a new controller is a single PR that creates `apps/<controller>.yaml` and (if it needs local manifests) a `controllers/<controller>/` chart.

## Per-cluster value injection

The gitops repo holds **zero** cluster-specific values. No account IDs, no role ARNs, no per-cluster `nodeSelector`s. Everything that varies between clusters arrives at sync time from a Kubernetes `Secret` labeled `argocd.argoproj.io/secret-type: cluster` in the `argocd` namespace.

The infrastructure repository creates that `Secret` from Terraform when ArgoCD is bootstrapped, populating its `config.values` field with facts that only Terraform can produce:

```
clusterName:                    eks-platform-dev-ap-northeast-1
awsRegion:                      ap-northeast-1
karpenterControllerRoleArn:     arn:aws:iam::<account>:role/...-karpenter-controller
karpenterNodeRoleName:          eks-platform-dev-ap-northeast-1-karpenter-node
karpenterInterruptionQueue:     eks-platform-dev-ap-northeast-1-karpenter-interruption
```

Each `ApplicationSet` in this repo uses ArgoCD's **cluster generator** to iterate over those `Secret`s and stamp out one `Application` per cluster, substituting `{{ .values.* }}` references at render time:

```yaml
spec:
  generators:
    - clusters:
        selector:
          matchLabels:
            argocd.argoproj.io/secret-type: cluster
  template:
    spec:
      source:
        helm:
          valuesObject:
            settings:
              clusterName: '{{ .values.clusterName }}'
              interruptionQueue: '{{ .values.karpenterInterruptionQueue }}'
            serviceAccount:
              annotations:
                eks.amazonaws.com/role-arn: '{{ .values.karpenterControllerRoleArn }}'
```

The pattern delivers three properties:

1. **No account-identifying values in Git.** ARNs and account IDs never enter the public repository.
2. **Multi-cluster by construction.** Adding a second cluster needs no change in this repo — the infrastructure registers a new cluster `Secret` and every `ApplicationSet` emits a new `Application` automatically.
3. **Reusable for every controller that needs IRSA.** Cilium, AWS LB Controller, External DNS, External Secrets, and the observability stack all consume the same `Secret`.

## Karpenter delivery

Karpenter ships as two cooperating `ApplicationSet`s with explicit sync ordering:

| Wave | File | Source | Purpose |
|---|---|---|---|
| 0 | `apps/karpenter.yaml` | Upstream Karpenter Helm chart (`public.ecr.aws/karpenter/karpenter`) | Controller, CRDs, webhook |
| 1 | `apps/karpenter-resources.yaml` | Local chart `controllers/karpenter/` | `EC2NodeClass` + `NodePool`s |

The split prevents the well-known race where `NodePool` resources are applied before the chart has installed the `NodePool` CRD. Each `ApplicationSet` also has its own prune / rollback envelope, so pausing or reverting resource definitions never destabilises the controller.

`NodePool` defaults — arm64-first Bottlerocket on Graviton, spot-first with on-demand fallback, aggressive consolidation, 30-day node expiry — are documented in `controllers/karpenter/README.md`.

## Adding a new controller

1. If the controller needs an IRSA role or any AWS resource, **add it in the infrastructure repo first**. Export the role ARN (and anything else cluster-specific) as a Terraform output. Extend the cluster `Secret` schema to carry the new value.
2. In this repo, create `apps/<controller>.yaml` — an `ApplicationSet` using the cluster generator, referencing the upstream chart and reading any cluster-specific value from `{{ .values.* }}`.
3. If the controller needs local CRDs or values templated with cluster-specific data, add `controllers/<controller>/` as a small local Helm chart.
4. Open a PR. ArgoCD picks up the new `ApplicationSet` on its next sync and renders the `Application`(s) into every registered cluster.

Chart-version bumps, NodePool tuning, and controller value tweaks need no infrastructure change — they flow through this repo and reconcile within the ArgoCD sync interval.

## Status

Initial scaffolding (Karpenter). Cilium, secrets/IRSA, RDS/ElastiCache adapters, the observability stack, and demo workloads follow.
