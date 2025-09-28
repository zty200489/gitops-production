# Production Kubernetes Cluster IaC Repository

This repository serves as the **source of truth** for production Kubernetes clusters IaC management powered by [FluxCD](https://fluxcd.io/) and [GitLab Agent for Kubernetes](https://docs.gitlab.com/ee/user/clusters/agent/).

All cluster configurations are version-controlled, reviewed, and automatically reconciled through pull-based continuous delivery, ensuring immutable infrastructure and audit-ready changes in production.

## Managed Services, Pods and Resources

### Infrastructure

#### Juicefs

[Juicefs](https://juicefs.com/zh-cn/) is a cloudnative high performance distributed file system. Flux manages:

- Pod (Helm): `juicefs-csi-controller`
- Pod (Helm): `juicefs-dashboard`
- Pod (Helm): `juicefs-csi-node`
- SC: `juicefs`
