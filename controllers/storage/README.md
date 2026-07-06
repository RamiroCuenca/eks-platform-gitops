# storage

Cluster `StorageClass` definitions backed by the EBS CSI driver, reconciled by
ArgoCD (`apps/storage.yaml`).

## What it ships

| Class | Backing | Notes |
|---|---|---|
| `gp3` | `ebs.csi.aws.com`, encrypted, baseline gp3 performance | `WaitForFirstConsumer`, expansion allowed, `Delete` reclaim |

The EBS CSI driver itself (managed addon + IRSA role) is installed by the
infra repo — it is a node-level storage primitive on the Terraform side of
the IaC↔GitOps boundary. The classes that ride on it iterate at application
cadence and belong here.

## Design notes

- **No cluster-default class.** Consumers reference `storageClassName: gp3`
  explicitly; an inherited default is an undeclared dependency with quiet
  failure modes.
- **`WaitForFirstConsumer`** so volumes are created in the AZ of the pod that
  mounts them — Immediate binding on a multi-AZ cluster can strand a volume
  where no consumer can schedule.
- **`Delete` reclaim** — the platform is build-and-destroy; `Retain` would
  orphan EBS volumes that silently accrue cost after teardown.

First consumers: Prometheus TSDB and Loki chunk storage
(`apps/kube-prometheus-stack.yaml`, `apps/loki.yaml`).
