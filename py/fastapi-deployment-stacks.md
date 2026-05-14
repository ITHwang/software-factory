# FastAPI Deployment Stacks

> Last updated: 2026-05-13

## TL;DR

How to deploy a FastAPI app to production — what each component (Uvicorn, Gunicorn, systemd, Nginx, ALB, ECS) actually owns, and which stack to pick by environment (local, EC2, ECS/Fargate).

**Use this when:**
- picking a production stack for a new service
- debugging why `--workers` alone isn't enough under load
- deciding between bare EC2 and ECS/Fargate
- auditing an existing deployment for missing supervision layers

**Don't use this for:**
- application-code patterns → `./python-backend-architecture.md`
- FastAPI request/response design → FastAPI's own docs
- container-image build best-practices (out of scope)

The doc names what each layer owns and does not prescribe a single stack — pick by the environment phase below.

## Table of Contents

| Phase | Section |
|---|---|
| 1. Concepts | [Quick Reference](#quick-reference), [WSGI vs ASGI](#wsgi-vs-asgi), [Component Roles](#component-roles) |
| 2. Stacks | [Local Development](#local-development), [Lightweight EC2](#lightweight-ec2), [Operationally Mature EC2](#operationally-mature-ec2), [ECS / Fargate](#ecs--fargate) |
| 3. Decide | [Picking A Stack](#picking-a-stack) |
| 4. Discipline | [Anti-Patterns](#anti-patterns) |

## Quick Reference

| Component | Role | Owns lifecycle of | Lives at |
|---|---|---|---|
| Uvicorn | ASGI server; runs the FastAPI app | the ASGI app (and worker processes when `--workers N`) | inside the process / container |
| Gunicorn | Process / worker supervisor (via `-k uvicorn_worker.UvicornWorker`) | Uvicorn worker processes (reload, timeout, recycle) | inside the process / container |
| systemd | OS-level service manager | the Python service process on a Linux host | bare VM / EC2 only |
| Nginx | Reverse proxy, TLS, buffering, static files | inbound HTTP traffic, TLS termination | bare VM / EC2 only |
| ALB | AWS-managed reverse proxy | inbound HTTP traffic, TLS, target-group routing | ECS / Fargate |
| ECS / Fargate | Container scheduler | container restart, health checks, rolling deploys, autoscaling | the orchestrator, not the container |

See also: [ASGI spec](https://asgi.readthedocs.io/en/latest/specs/main.html), [Uvicorn deployment](https://www.uvicorn.org/deployment/), [Gunicorn design](https://docs.gunicorn.org/en/stable/design.html), [FastAPI deployment concepts](https://fastapi.tiangolo.com/deployment/concepts/).

## WSGI vs ASGI

Python web apps don't speak HTTP directly. A server in front of the app translates HTTP into a standardized call into Python code. **WSGI** (Web Server Gateway Interface) is the older synchronous, request-response standard — used by Flask and Django's classic stack. **ASGI** (Asynchronous Server Gateway Interface) is the modern async successor — it supports `async`/`await`, WebSockets, streaming, and long-lived connections.

```text
WSGI stack                        ASGI stack
Client                            Client
  ↓ HTTP                            ↓ HTTP / WebSocket
Gunicorn / uWSGI                  Uvicorn
  ↓ WSGI                            ↓ ASGI
Flask / Django (WSGI app)         FastAPI / Starlette (ASGI app)
```

FastAPI is an **ASGI** app. Gunicorn's default sync worker (`-k sync`) is WSGI and cannot run it — that's why production stacks pair Gunicorn with an explicit ASGI worker class (`uvicorn_worker.UvicornWorker`).

See also: [ASGI spec](https://asgi.readthedocs.io/en/latest/specs/main.html), [Uvicorn ASGI concepts](https://www.uvicorn.org/concepts/asgi/).

## Component Roles

Each piece in the stack owns one concern. Knowing what it *doesn't* own is as important as knowing what it does.

- **Uvicorn** — the ASGI server. Runs the FastAPI app inside one process. Since Uvicorn 0.30+, native multi-worker support is built in: `--workers N` forks N worker processes that each load the app. This is now the modern default for "I just want to use my CPU cores." Uvicorn does **not** own graceful reload semantics, request-count recycling, or boot-time service supervision — those belong to Gunicorn or the orchestrator.
- **Gunicorn** — a process and worker supervisor. With `-k uvicorn_worker.UvicornWorker` it supervises Uvicorn ASGI workers. Owns: worker spawn/restart, graceful reload, request timeouts, `--max-requests` recycling, signal handling. Does **not** own ASGI itself (that's the worker class) and does **not** replace OS-level service supervision (that's systemd / the orchestrator).
- **systemd** — Linux service manager. Owns boot-time startup, restart-on-crash, log routing (`journalctl`), and PID management for a long-lived Python service. Only relevant on bare VMs (EC2). In a container orchestrator, the orchestrator owns this — running systemd inside an ECS task is an anti-pattern.
- **Nginx** — reverse proxy on a VM. Owns TLS termination, request buffering, static-file serving, connection limits, header rewriting (`X-Forwarded-*`). Only relevant on bare VMs — in ECS, ALB replaces it.
- **ALB** — AWS-managed reverse proxy. Owns TLS, HTTP/2, target-group routing, host/path rules, health-check polling. Replaces Nginx in ECS/Fargate.
- **ECS / Fargate** — container scheduler. Owns container restart, desired-count enforcement, rolling deploys, autoscaling, health checks. Does **not** look inside the container; if a worker hangs but the process stays alive, ECS will happily call the task healthy.

**Invariant: container orchestration ≠ worker orchestration.** ECS keeps a container running; Gunicorn (or Uvicorn's own timeout config) keeps individual workers honest. Both are needed for an operationally sensitive service.

See also: [Uvicorn deployment](https://www.uvicorn.org/deployment/), [Gunicorn design](https://docs.gunicorn.org/en/stable/design.html).

## Local Development

Run Uvicorn directly with autoreload:

```bash
uvicorn app.main:app --reload
```

No Nginx, no Gunicorn, no systemd. `--reload` watches the source tree and bounces the process on edit. This is the only stack where a single Uvicorn process is fine — `--reload` is explicitly not for production.

See also: [Uvicorn settings](https://www.uvicorn.org/settings/).

## Lightweight EC2

Stack: **Nginx + systemd + `uvicorn --workers N`**. The modern default for small-to-mid production services on a bare VM.

```bash
uvicorn app.main:app --host 127.0.0.1 --port 8000 --workers 4
```

```text
Client
  ↓ HTTPS
Nginx          (TLS, buffering, static files)
  ↓ HTTP (loopback)
systemd        (service supervision)
  ↓
Uvicorn workers (×N)
  ↓ ASGI
FastAPI
```

Best simplicity/performance tradeoff when graceful-reload, per-request timeouts, and worker recycling aren't operational requirements. If they are, jump to the next tier.

See also: [FastAPI server workers](https://fastapi.tiangolo.com/deployment/server-workers/), [Uvicorn deployment](https://www.uvicorn.org/deployment/), [FastAPI behind a proxy](https://fastapi.tiangolo.com/advanced/behind-a-proxy/).

## Operationally Mature EC2

Stack: **Nginx + systemd + Gunicorn + Uvicorn worker**. The upgrade tier when worker-level discipline matters.

```bash
gunicorn app.main:app \
  -k uvicorn_worker.UvicornWorker \
  --workers 4 --bind 127.0.0.1:8000 \
  --timeout 60 --graceful-timeout 30 \
  --max-requests 1000 --max-requests-jitter 100
```

```text
Client
  ↓ HTTPS
Nginx          (TLS, buffering, static files)
  ↓ HTTP (loopback)
systemd        (service supervision)
  ↓
Gunicorn master (worker supervision)
  ↓
Uvicorn workers (×N)
  ↓ ASGI
FastAPI
```

Why add Gunicorn at all when Uvicorn now has `--workers`? Because Gunicorn brings:

- **Graceful reload** on `SIGHUP` — drain in-flight requests before swapping workers.
- **Per-request timeout** (`--timeout`) — kill a worker stuck on a slow upstream.
- **Worker recycling** (`--max-requests`, `--max-requests-jitter`) — periodic process restart that contains memory leaks and keeps tail latency stable.
- **Mature signal handling** for zero-downtime deploys.

The worker class lives in the separate `uvicorn-worker` package (PyPI, import `uvicorn_worker.UvicornWorker`). The legacy `uvicorn.workers.UvicornWorker` from inside the `uvicorn` package still works but is no longer canonical — new projects should depend on `uvicorn-worker` directly and use the new import path.

See also: [Gunicorn design](https://docs.gunicorn.org/en/stable/design.html), [`uvicorn-worker` on PyPI](https://pypi.org/project/uvicorn-worker/), [FastAPI server workers](https://fastapi.tiangolo.com/deployment/server-workers/).

## ECS / Fargate

Stack: **ALB + ECS/Fargate + (`uvicorn --workers N` *or* Gunicorn + Uvicorn worker)**. Two layers from the EC2 stack disappear and two new owners take over.

- **ECS** owns container restart, desired count, health-check polling, rolling deploys, autoscaling — so **systemd disappears**.
- **ALB** owns TLS, HTTP/2, target-group routing, host/path rules — so **Nginx usually disappears**.
- **Inside the container** you still need a worker supervisor. Two viable choices: bare `uvicorn --workers N` (simpler, fewer moving parts) or `gunicorn -k uvicorn_worker.UvicornWorker` (richer supervision, see previous section). Pick based on operational sensitivity, not container count.

```text
Client
  ↓ HTTPS
ALB                          (TLS, target-group routing)
  ↓ HTTP
ECS / Fargate task           (container restart, health checks, autoscaling)
  ↓
Gunicorn  OR  Uvicorn        (worker supervision inside the container)
  ↓
Uvicorn workers (×N)
  ↓ ASGI
FastAPI
```

The trap: ECS health checks answer "is the container's port answering?" — not "are the workers inside healthy?" A leaking, looping, or wedged worker can pass an HTTP health check while serving real traffic poorly. Worker-level supervision (Gunicorn's `--timeout` / `--max-requests`, or careful Uvicorn timeout settings) is what catches this.

See also: [AWS ECS task definition parameters](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_definition_parameters.html), [AWS ALB docs](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/), [FastAPI Docker](https://fastapi.tiangolo.com/deployment/docker/).

## Picking A Stack

| Environment | External traffic | Service supervision | Worker supervision | App server |
|---|---|---|---|---|
| Local dev | — | — | — | `uvicorn --reload` |
| Lightweight EC2 | Nginx | systemd | (none — `--workers` flag) | `uvicorn --workers N` |
| Mature EC2 | Nginx | systemd | Gunicorn | `gunicorn -k uvicorn_worker.UvicornWorker` |
| ECS/Fargate (simple) | ALB | ECS | (none — `--workers` flag) | `uvicorn --workers N` |
| ECS/Fargate (mature) | ALB | ECS | Gunicorn | `gunicorn -k uvicorn_worker.UvicornWorker` |

The progression is always the same: one owner per lifecycle scope. **External traffic** lives at Nginx or ALB. **Service supervision** lives at systemd (VM) or ECS (container). **Worker supervision** is the cell that changes most — start without it, add Gunicorn when graceful reloads, timeouts, or worker recycling become operational requirements.

See also: [FastAPI deployment concepts](https://fastapi.tiangolo.com/deployment/concepts/).

## Anti-Patterns

- **Don't use `tiangolo/uvicorn-gunicorn-fastapi`.** The image is officially deprecated by FastAPI's own docker page (`https://hub.docker.com/r/tiangolo/uvicorn-gunicorn` — listed here only for reference, do not depend on it). Build your own slim image with a plain `CMD` of either `fastapi run --workers N` or `gunicorn -k uvicorn_worker.UvicornWorker`.
- **Don't run a single Uvicorn process behind ALB and expect it to scale.** A bare `uvicorn app.main:app` uses one CPU core regardless of how many vCPUs the task has. Pass `--workers N` (or use Gunicorn) so multiple workers share the load.
- **Don't stack systemd inside an ECS container.** ECS already owns the container's lifecycle. Use the container's `CMD` directly and let ECS restart on exit.
- **Don't trust container health checks alone.** They answer "is the process alive?", not "are the workers healthy?" Add Gunicorn `--timeout` and `--max-requests` supervision, or set Uvicorn `--timeout-keep-alive` and `--timeout-graceful-shutdown` deliberately.
- **Don't use Gunicorn's default sync worker (`-k sync`) for FastAPI.** That's a WSGI worker; it cannot run an ASGI app. Always specify `-k uvicorn_worker.UvicornWorker`.
- **Don't mix legacy `uvicorn.workers.UvicornWorker` with the new `uvicorn-worker` package** in the same project. Pick one — new projects should use the `uvicorn-worker` package and the `uvicorn_worker.UvicornWorker` import path.

See also: [FastAPI Docker](https://fastapi.tiangolo.com/deployment/docker/), [FastAPI server workers](https://fastapi.tiangolo.com/deployment/server-workers/), [Gunicorn design](https://docs.gunicorn.org/en/stable/design.html).
