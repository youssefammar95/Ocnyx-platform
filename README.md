# ocnyx-platform

Infrastructure and deployment platform for Ocnyx, a FastAPI/React IoT SaaS.

This project takes Ocnyx from a PaaS (Railway) to a reproducible AWS/Kubernetes platform, built step by step as a hands-on DevOps learning project. Production stays on Railway; this repo is a staging and learning environment.

Along the way it fixes a real architectural limitation: the API and background workers ran in one process, which prevented horizontal scaling. They are being separated via `OCNYX_ROLE=api|worker|all`.

## Status

**M1 — Docker: in progress**

## Roadmap

| # | Milestone | Goal | Status |
|---|-----------|------|--------|
| M1 | Docker | Full stack runs locally with `docker compose up` | 🚧 |
| M2 | Kubernetes (local) | Same system on `kind`/`k3d`, packaged as a Helm chart | ⏳ |
| M3 | Terraform + AWS | EKS cluster that can be created and destroyed on demand | ⏳ |
| M4 | Jenkins | Push to Ocnyx deploys automatically to the cluster | ⏳ |
| M5 | Ansible | Jenkins server provisioned reproducibly | ⏳ |
| M6 | Monitoring | Prometheus/Grafana, alerts, HPA demo under load | ⏳ |

## Tech Stack

Docker · Docker Compose · Kubernetes · Helm · Terraform · AWS (EKS, ECR, VPC) · Jenkins · Ansible · Prometheus · Grafana

## Repository Layout

```
docs/
  learnings.md   # what went wrong, why, and how it was fixed
  adr/           # architecture decision records, one per milestone
```

More folders (`docker/`, `k8s/`, `helm/`, `terraform/`, `jenkins/`, `ansible/`) are added as each milestone starts.

## Security

No secrets are stored in this repository. Credentials are injected at runtime.
