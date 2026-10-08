# Learnings

What went wrong, why, and how I fixed it.

## M1 — Docker

### uvicorn is not PID 1
**Observation:** The log says `Started server process [7]`. Inside the container, PID 1 is `sh -c "uvicorn ..."` and uvicorn is its child.
**Why:** The `CMD` uses `sh -c` to expand `${PORT}`, so the shell becomes PID 1.
**Why it matters:** `docker stop` sends SIGTERM to PID 1 only. If the shell does not forward it, uvicorn never gets the signal, Docker kills it after 10 s and the graceful shutdown in `lifespan` (closing MQTT and websockets) does not run.
**Evidence:** The stopped `before` container shows `Exited (137)`, i.e. it was killed with SIGKILL, not shut down cleanly.
**Fix:** `CMD ["sh", "-c", "exec uvicorn ..."]`. The shell still expands `${PORT}`, then `exec` replaces itself with uvicorn, so uvicorn becomes PID 1.
**Result:**

| | before | after |
|---|---|---|
| Log | `Started server process [7]` | `Started server process [1]` |
| `docker stop` | ~10 s, `Exited (137)` (SIGKILL) | 2.4 s, `Exited (0)` |
| Graceful shutdown | did not run | `Application shutting down` → `[MQTT] Client stopped` → `All websockets closed` → `Application shutdown complete` |

The 2.4 s are real cleanup work (stopping the MQTT client alone takes ~1 s), not waiting for a timeout.

### Multi-stage build for the Python packages
**Problem:** The runtime image contained `gcc`/`g++`, which are only needed while `pip install` compiles packages.
**Fix:** A separate `builder` stage installs the compilers and runs `pip install` into a virtualenv (`/opt/venv`). The final stage starts from a clean `python:3.11-slim` and copies only `/opt/venv`. `PATH` has to be set again in the final stage, because every stage starts from scratch.
**Result:**

| Image | Disk usage | Content size |
|---|---|---|
| `before` (original Dockerfile) | 1.25 GB | 293 MB |
| `after` (final Dockerfile) | 897 MB | 200 MB |

About −28 % on disk and −32 % content size.

### Running as a non-root user
**Problem:** The app ran as `root` inside the container. A vulnerability in the app would give an attacker full rights in the container.
**Fix:**
```dockerfile
RUN useradd --create-home --uid 1000 ocnyx \
    && mkdir -p models \
    && chown ocnyx:ocnyx models
USER ocnyx
```
**Result:** `whoami` in the container returns `ocnyx`, and `models/` is owned by `ocnyx:ocnyx`.

### HEALTHCHECK
**Goal:** Docker should know whether the app actually works, not only whether the process is running.
**Fix:**
```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --start-period=30s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/api/health', timeout=4)" || exit 1
```
**Result:** `docker ps` shows `(health: starting)`, then `(healthy)` after about 10 s.

### Containers find each other by service name, not `localhost`
**Problem:** Inside a container, `localhost` is the container itself, not the host and not other containers.
**Fix:** Compose puts all services into one network where each service is reachable by its name: `MQTT_BROKER=emqx`, `OCNYX_BASE_URL=http://api:8000`.
**Ports:** In `"8000:8000"` the left side is the host port (for my browser), the right side is the container port. Containers talk to each other on the container port; `ports:` is only needed for access from outside.
