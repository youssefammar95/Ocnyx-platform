# ocnyx-platform

Infrastructure and deployment platform for Ocnyx, a FastAPI/React IoT SaaS.

This project takes Ocnyx from a PaaS (Railway) to a reproducible AWS/Kubernetes platform, built step by step as a hands-on DevOps learning project. Production stays on Railway; this repo is a staging and learning environment.

Along the way it fixes a real architectural limitation: the API and background workers ran in one process, which prevented horizontal scaling. They are now separated via `OCNYX_ROLE=api|worker|all` ([ADR 001](docs/adr/001-docker-compose-api-worker-split.md)).

## Status

**M1 — Docker: done** · next: M2 — Kubernetes (local)

## Roadmap

| # | Milestone | Goal | Status |
|---|-----------|------|--------|
| M1 | Docker | Full stack runs locally with `docker compose up` | ✅ |
| M2 | Kubernetes (local) | Same system on `kind`/`k3d`, packaged as a Helm chart | ⏳ |
| M3 | Terraform + AWS | EKS cluster that can be created and destroyed on demand | ⏳ |
| M4 | Jenkins | Push to Ocnyx deploys automatically to the cluster | ⏳ |
| M5 | Ansible | Jenkins server provisioned reproducibly | ⏳ |
| M6 | Monitoring | Prometheus/Grafana, alerts, HPA demo under load | ⏳ |

## Quick Start

Requires the app repo checked out next to this one (`../Ocnyx`) and a `.env` with the Supabase and API keys.

```bash
docker compose up --build --scale api=2
```

Dashboard and API: http://localhost:8000 · EMQX dashboard: http://localhost:18083

## Tech Stack

Docker · Docker Compose · Kubernetes · Helm · Terraform · AWS (EKS, ECR, VPC) · Jenkins · Ansible · Prometheus · Grafana

## Repository Layout

```
compose.yml       # local stack: emqx, redis, api, worker, proxy, simulator
docker/
  Dockerfile     # multi-stage image for api and worker
  nginx.conf     # load balancer for the api replicas
docs/
  learnings.md   # what went wrong, why, and how it was fixed
  adr/           # architecture decision records, one per milestone
```

More folders (`k8s/`, `helm/`, `terraform/`, `jenkins/`, `ansible/`) are added as each milestone starts.

## Security

No secrets are stored in this repository. Credentials are injected at runtime.
