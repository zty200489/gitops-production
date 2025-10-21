# Production Kubernetes Cluster IaC Repository

This repository serves as the **source of truth** for production Kubernetes clusters IaC management powered by [FluxCD](https://fluxcd.io/) and [GitLab Agent for Kubernetes](https://docs.gitlab.com/ee/user/clusters/agent/).

All cluster configurations are version-controlled, reviewed, and automatically reconciled through pull-based continuous delivery, ensuring immutable infrastructure and audit-ready changes in production.

## Managed Services, Pods and Resources

### Infrastructure

#### Cert Manager

[Cert Manager](https://cert-manager.io/) is a a powerful and extensible X.509 certificate controller for Kubernetes and OpenShift workloads. It will obtain certificates from a variety of Issuers, both popular public Issuers as well as private Issuers, and ensure the certificates are valid and up-to-date, and will attempt to renew certificates at a configured time before expiry.

Cert Manager was donated to [**CNCF**](https://www.cncf.io/projects/cert-manager/) in 2020, moved to the Incubating maturity level on September 19, 2022, and then moved to the Graduated maturity level on September 29, 2024.

Flux manages the following resources under the namespace `cert-manager`:

- HelmRelease: `cert-manager`
- ClusterIssuer: `self-signed-issuer`

#### Democratic CSI

[Democratic CSI](https://github.com/democratic-csi/democratic-csi) is a CSI solution for container orchestration systems. It primarily supports integration with ZFS backend storage solutions (e.g. TrueNAS) via NFS for iSCSI remote file/block stroage protocals, but also supports many other clients. Since we already have `juicefs` as a distributed remote storage solution, we use ZFS and iSCSI to create efficient block storage for low latency random read-write tasks (e.g. database).

Democratic CSI is a **community maintained project** but works exceptionally well. 

Flux manages the following resources under the namespace `storage`:

- HelmRelease: `democratic-csi`
- StorageClass: `iscsi`

#### Juicefs

[Juicefs](https://juicefs.com/en/) ([中文站](https://juicefs.com/zh-cn/)) is a cloudnative high performance, cloudnative, distributed file system.

JuiceFS is a open-source project maintained by **Juicedata, Inc**. The company profits through enterprise SaaS, and is actively maintaining the project.

Flux manages the following resources under the namespace `juicefs`:

- HelmRelease: `juicefs-csi-driver`
- StorageClass: `juicefs-0`
- PodMonitor: `juicefs-mounts-monitor`
- ConfigMap: `juicefs-dashboard`

#### Prometheus Stack

[Kube Prometheus Stack](https://prometheus-operator.dev/) is a prometheus operator for kubernetes.

Kube Prometheus Stack is a **jointly maintained community project** that adapts prometheus for kubernetes using operators. It is an abstaction atop the existing popular monitoring solution including [Prometheus](https://prometheus.io/) and [Grafana](https://grafana.com/grafana/), which themselves are maintained by [Grafana Labs](https://grafana.com/). The company profits through enterprice SaaS, and is actively maintaining the projects.

Flux manages the following resources under the namespace `kube-prometheus`:

- HelmRelease: `kube-prometheus-stack`

#### Cloud Native PostgreSQL

[Cloud Native PostgreSQL](https://cloudnative-pg.io/) is a cloud native solution for running postgresql in kubernetes using operators. It offeres native WAL streaming replication, promary/standby clustering, and many more features.

CNPG was originally created by [EDB](https://www.enterprisedb.com/products/edb-postgres-ai-for-cloudnativepg) and was accepted to [CNCF](https://www.cncf.io/projects/cloudnativepg/) on January 21, 2025 at the Sandbox maturity level.

Flux manages the following resources under the namespace `cnpg-system`:

- HelmRelease: `cnpg`

**Remark:** When creating clusters, be sure to add custom labels `zty89.com/prometheus: default-prometheus` if `.spec.monitoring.enablePodMonitor` was set to `true`.
