# Production Kubernetes Cluster IaC Repository

This repository serves as the **source of truth** for production Kubernetes clusters IaC management powered by [FluxCD](https://fluxcd.io/) and [GitLab Agent for Kubernetes](https://docs.gitlab.com/ee/user/clusters/agent/).

All cluster configurations are version-controlled, reviewed, and automatically reconciled through pull-based continuous delivery, ensuring immutable infrastructure and audit-ready changes in production.

## Managed Services, Pods and Resources

### Infrastructure

#### Democratic CSI

[Democratic CSI](https://github.com/democratic-csi/democratic-csi) is a CSI solution for container orchestration systems. It primarily supports integration with ZFS backend storage solutions (e.g. TrueNAS) via NFS for iSCSI remote file/block stroage protocals, but also supports many other clients. Since we already have `juicefs` as a distributed remote storage solution, we use ZFS and iSCSI to create efficient block storage for low latency random read-write tasks (e.g. database). Flux manages under namespace `storage`:

- HelmRelease: `democratic-csi`
- StorageClass: `iscsi`

#### Juicefs

[Juicefs](https://juicefs.com/zh-cn/) is a cloudnative high performance distributed file system. Flux manages under namespace `juicefs`:

- HelmRelease: `juicefs-csi-driver`
- StorageClass: `juicefs-0`
- PodMonitor: `juicefs-mounts-monitor`
- ConfigMap: `juicefs-dashboard`

#### Prometheus Stack

[Kube Prometheus Stack](https://prometheus-operator.dev/) is a prometheus operator for kubernetes. Flux manages under namespace `kube-prometheus`:

- HelmRelease: `kube-prometheus-stack`

#### Cloud Native PostgreSQL

[Cloud Native PostgreSQL](https://cloudnative-pg.io/) is a cloud native solution for running postgresql in kubernetes. It offeres native WAL streaming replication, promary/standby clustering, and many more features. Flux manages under namespace `cnpg-system`:

- HelmRelease: `cnpg`

**Remark:** When creating clusters, be sure to create a custom `PodMonitor` for that cluster alongside it. Default setting `.spec.monitoring.enablePodMonitor` does not support custom labels, which failed to be automatically picked up by the prometheus operator.
