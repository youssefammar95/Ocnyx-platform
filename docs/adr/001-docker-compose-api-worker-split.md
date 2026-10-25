# ADR 001 — Local stack with Docker Compose and an api/worker split

**Status:** Accepted · **Milestone:** M1 — Docker · **Date:** 2026-10-25

## Context
Ocnyx ran as one process on Railway: HTTP API, MQTT ingestion and five background schedulers (anchoring, inactivity, feedback retrain, retrain, data retention) in the same `uvicorn` process. Running a second instance would duplicate the MQTT connection (same client ID → the broker disconnects one of them) and run every scheduled job twice. The API could not be scaled horizontally.

M1 needs the full stack to run locally with one command, and it should already have the shape needed for Kubernetes in M2.

## Decision
- **One image, several roles.** The app reads `OCNYX_ROLE=api|worker|all` (default `all`, so Railway keeps working unchanged). `api` serves HTTP and websockets only; `worker` runs MQTT ingestion and the schedulers. An invalid value fails at startup instead of silently falling back to `all`.
- **Redis for websocket fan-out.** The worker publishes live updates to the Redis channel `ocnyx:ws:customer`; every API replica subscribes and forwards them to its own websocket clients.
- **Compose services:** `emqx` (MQTT broker), `redis`, `api` (scalable), `worker` (exactly one), `proxy` (nginx on port 8000, load-balances the api replicas incl. websockets), `simulator` (sends telemetry over MQTT).
- **Image:** multi-stage build (Node for the dashboard, a builder stage for Python wheels, slim runtime), non-root user, `HEALTHCHECK` on `/api/health`, uvicorn as PID 1 via `exec`.
- **Secrets** come from a local, git-ignored `.env`; nothing is baked into the repo.

## Consequences
- `docker compose up --build --scale api=N` runs N API replicas without duplicate jobs or MQTT conflicts (see `docs/learnings.md`).
- The worker is a singleton. Scaling it would need leader election or a job queue — out of scope for now.
- The worker still runs uvicorn so it can reuse the same `HEALTHCHECK`; it serves no external traffic.
- Redis becomes a required dependency whenever `api` and `worker` run in separate processes.
- The same split maps directly to M2: a `Deployment` for `api` (replicas + `Service` instead of nginx), a single-replica `Deployment` for `worker`.
