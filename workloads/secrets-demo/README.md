# secrets-demo

Demonstration workload for the platform's secret-management model: an
application reads an **AWS Secrets Manager** secret through a **per-workload
IRSA role** scoped to that one secret, delivered by the **AWS Secrets and
Configuration Provider (ASCP)** as a mounted file.

## What it deploys

| Resource | Purpose |
|---|---|
| `ServiceAccount/demo-app` | Annotated with `eks.amazonaws.com/role-arn` (the Terraform-created IRSA role). |
| `SecretProviderClass/demo-app-secrets` | Tells the AWS provider which secret to fetch (`objectType: secretsmanager`). |
| `Deployment/demo-app` | Single pod that mounts the secret as a file and otherwise sleeps. |

All three land in the **`demo`** namespace, which is created and governed by the
`network-policies` chart (default-deny egress + a DNS allow).

## The point: a zero-egress pod still gets its secret

The workload pod makes **no network call to AWS**. The kubelet → CSI driver →
AWS provider handoff is a node-local socket; the **AWS provider DaemonSet** (in
`kube-system`, outside the default-deny scope) performs the IRSA-scoped
`GetSecretValue` and hands the pod a file. So this pod runs under strict
default-deny egress (DNS only) and still receives its secret; no Secrets
Manager allow-rule is needed in the app namespace.

The value is mounted on **tmpfs** (memory) for the pod's lifetime and is never
written to a Kubernetes Secret in etcd; `syncSecret` is intentionally off.

## Values

Injected at sync time from the ArgoCD cluster Secret (see `apps/secrets-demo.yaml`):

| Value | Source annotation |
|---|---|
| `demoAppSecretsRoleArn` | `platform.io/demo-app-secrets-role-arn` |
| `demoSecretName` | `platform.io/demo-secret-name` |
| `awsRegion` | `platform.io/aws-region` |

Nothing account-specific is committed here; the role ARN and secret name arrive
through the cluster Secret the infra repo's `argocd` module writes.

## Verify

```bash
# The pod schedules onto a Karpenter node (it has no system-tier toleration),
# which is also the Karpenter scale-up demo.
kubectl -n demo get pods -l app=demo-app

# Read the mounted secret (exec is node-local, so it works under default-deny).
kubectl -n demo exec deploy/demo-app -- cat /mnt/secrets-store/eks-platform/<env>/demo-app/credentials
```
