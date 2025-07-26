# Homelab GitOps: Kubernetes Applications

This repository contains the Kubernetes manifests and configurations to deploy a full suite of applications onto a homelab cluster. It follows the GitOps methodology, using ArgoCD as the engine to ensure the state of the live cluster matches the declarative configuration in this repository.

This project is the second part of a two-part homelab setup, intended to be used after the infrastructure has been provisioned by the homelab-iac repository.

## Core Concepts

This repository is built on the "App of Apps" pattern, a powerful and scalable way to manage multiple applications with ArgoCD.

- **Bootstrap**: A single "root" ArgoCD application (`bootstrap/root.yaml`) is applied to the cluster.
- **App of Apps**: This root application's only job is to deploy other ArgoCD Application resources. These are defined in the `apps/system/` directory.
- **Sync Waves**: Each application is assigned a sync-wave annotation. This enforces a strict order of deployment, ensuring that dependencies (like namespaces, shared configurations, and secrets operators) are created before the applications that rely on them.
- **Declarative Configuration**: The entire application stack—including the monitoring suite, storage, and dashboards—is defined as code. Secrets are managed securely and are never stored in Git.

## Deployed Services

| Service | URL | Namespace | Description |
|---------|-----|-----------|-------------|
| Homer | https://homer.milenika.dev | homer | A simple, static homepage dashboard. |
| ArgoCD | https://argocd.milenika.dev | argocd | The GitOps engine managing the cluster. |
| Grafana | https://grafana.milenika.dev | monitoring | Visualization for metrics, logs, and traces. |
| MinIO | https://minio.milenika.dev | minio | S3-compatible object storage console. |
| MinIO API | https://minio-api.milenika.dev | minio | S3 API endpoint for applications. |
| Hugo Site | https://hugo.milenika.dev | nginx | Example static site served by NGINX from MinIO. |
| Loki | (Available in Grafana) | monitoring | Log aggregation system. |
| Tempo | (Available in Grafana) | monitoring | Distributed tracing backend. |
| Prometheus | (Available in Grafana) | monitoring | Metrics collection and alerting. |

## Prerequisites

Before applying the manifests in this repository, you must have:

- **A running Kubernetes cluster**: This setup is designed for the K3s cluster provisioned by the homelab-iac repository.
- **Functional kubectl**: You need kubectl configured to communicate with your cluster.
- **ArgoCD Installed**: ArgoCD must be installed in the argocd namespace. The homelab-iac setup includes an ArgoCD bootstrap module (07_argocd_bootstrap).
- **External Secrets Operator (ESO)**: The homelab-iac repo also deploys ESO, which is critical for secret management.

## How to Deploy

The entire application stack can be bootstrapped with a single command. This applies the "root" application, which then instructs ArgoCD to deploy everything else.

```bash
kubectl apply -f bootstrap/root.yaml
```

After applying, you can watch the applications sync and become healthy in the ArgoCD UI at https://argocd.milenika.dev.

## Repository Structure

```
.
├── apps/
│   ├── external-secrets/ # ExternalSecret definitions for fetching secrets
│   ├── minio/            # Helm values for MinIO object storage
│   ├── monitoring/       # The entire LGTM stack (Loki, Grafana, Tempo, Prometheus)
│   │   ├── dashboards/   # Declarative Grafana dashboards
│   │   └── namespace/    # Namespace, RBAC, and Middleware for monitoring
│   ├── nginx/            # Helm values for the NGINX static site server
│   └── system/           # ArgoCD Application manifests (The "App of Apps")
└── bootstrap/
    └── root.yaml         # The single entry point to bootstrap the entire cluster
```

## Secret Management

**No secrets are stored in this repository.**

This project uses the External Secrets Operator (ESO) pattern.

- Actual secret values (API keys, passwords) are stored securely in Google Secret Manager (provisioned by the homelab-iac repo).
- The YAML files in `apps/external-secrets/` are ExternalSecret resources.
- These resources tell ESO: "Go to Google Secret Manager, fetch the secret named grafana-admin-password, and create a standard Kubernetes Secret named grafana-admin-secret in the monitoring namespace."
- Applications are then configured to use these Kubernetes Secret resources, completely decoupling them from the secret management backend.

## Troubleshooting

```bash
# Check the sync status of the root application
kubectl get app root -n argocd -o yaml

# Get the status of all managed applications
kubectl get applications -n argocd

# Force ArgoCD to refresh an application (if it seems stuck)
argocd app get prometheus-stack --refresh

# See the logs for the ArgoCD Application Controller
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller
```