# Celery Lessons

A reference collection of guides and patterns for building production-grade Celery task pipelines in Python 3.12+. The materials focus on **durable, idempotent execution** that survives worker crashes, pod evictions, and network partitions without losing work or producing duplicate side effects.

## FastAPI alignment (recommended)

In our production stacks, Celery workers typically run behind a FastAPI service (FastAPI is the "backend" / API layer that publishes tasks; Celery is the durable execution layer that performs them). When you use these patterns in a FastAPI + Celery system, we recommend following the organization and engineering practices from **zhanymkanov/fastapi-best-practices** and applying them consistently across the FastAPI app *and* the Celery code:

- **Project structure**: organize by domain/module (keep task definitions, schemas, and service code close together).
- **Configuration**: prefer environment-driven configuration (e.g., `pydantic-settings`) for broker URLs, queue names, timeouts, and feature flags.
- **Dependencies**: keep dependency boundaries clear (shared Pydantic models/utilities in a common module; avoid circular imports between API and tasks).

---

## Contents

| File | Description |
|------|-------------|
| [`Celery Durable Execution Queue Pattern.md`](./Celery%20Durable%20Execution%20Queue%20Pattern.md) | End-to-end reference for the four-layer durable execution pattern — broker configuration, distributed locking, idempotency checkpoints, chain/chord durability, Kubernetes and ECS deployment notes, and a full anti-pattern catalogue. |
| [`Celery Agent.md`](./Celery%20Agent.md) | Coding-agent rules and standards for Python 3.12+ Celery development: type-safety requirements, Pydantic data-model guidance, security standards, and a pre-submission checklist. |

---

## Lessons Covered

### Celery Durable Execution Queue Pattern
- **Core concepts** — what "durable execution" means and the three failure modes to design against (worker death before ACK, racing workers, lost ACK)
- **Broker / worker configuration** — required `celery_app.py` settings (`task_acks_late`, `task_reject_on_worker_lost`, `worker_prefetch_multiplier`, `visibility_timeout`, etc.)
- **The four-layer durable execution pattern**
  1. Idempotency pre-check — detect already-completed work and return early
  2. Distributed critical-section lock (mutex) — prevent racing workers
  3. Per-step completion checkpoints — resume after partial failures
  4. Retry instead of Ignore — never silently drop contested messages
- **Distributed lock implementation** — Redis/Valkey Lua-based lock helper (`DistributedLock`)
- **Full task skeleton** — copy-paste pattern for any non-idempotent task
- **Chain / chord durability** — keeping multi-step pipelines durable
- **Kubernetes-specific settings** — EKS/GKE/AKS pod lifecycle considerations
- **ECS-specific settings** — Fargate / EC2 task lifecycle considerations
- **Anti-patterns catalogue** — common mistakes and their correct alternatives

### Celery Agent (Python Coding Standards)
- Modern Python 3.12+ type annotations (`str | None`, no `Optional`/`Union`)
- When to use `dict[str, primitive]` versus Pydantic `BaseModel`
- Forbidden patterns: bare `dict`, `Any`, magic-string dict access, `# pyright: ignore`
- Security standards — Pydantic input validation, `pydantic-settings` for secrets, safe structured logging with `structlog`
- Celery-specific checklist integrated into the development workflow

---

## Prerequisites

- **Python 3.12+**
- **Redis or Valkey** as both the Celery broker and result backend
- Python packages referenced in the guides:
  - `celery`
  - `redis`
  - `pydantic`
  - `pydantic-settings`
  - `structlog`

> The guides assume a Redis/Valkey broker. No other broker (RabbitMQ, SQS, etc.) is covered.

---

## Setup

```bash
# 1. Create and activate a virtual environment
python3 -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate

# 2. Install the referenced packages
pip install celery redis pydantic pydantic-settings structlog
```

Start Redis locally (Docker is the quickest path):

```bash
docker run -d -p 6379:6379 redis:latest
```

---

## Using the Guides

The files in this repository are reference documents, not runnable code. Read them in the following order:

1. **[`Celery Durable Execution Queue Pattern.md`](./Celery%20Durable%20Execution%20Queue%20Pattern.md)** — understand the pattern and copy the configuration/skeleton into your project.
2. **[`Celery Agent.md`](./Celery%20Agent.md)** — apply the coding standards and use the pre-submission checklist when writing or reviewing tasks.

---

## Repository Structure

```
Celery-Lessons/
├── README.md                                      ← this file
├── Celery Agent.md                                ← Python/Celery coding standards & agent rules
└── Celery Durable Execution Queue Pattern.md      ← Four-layer durable execution reference guide
```

---

## Notes & Common Gotchas

| Gotcha | Explanation |
|--------|-------------|
| `visibility_timeout` too short | If `visibility_timeout` is shorter than your longest task's wall-clock time, the broker re-delivers the live task to a second worker, causing duplicate execution. Always set it longer than your worst-case task runtime. |
| Using `raise Ignore()` for lock contention | `Ignore()` permanently discards the message. Contention on a lock means *another worker is running*, not that the work is done — always use `raise self.retry(countdown=…)` instead. |
| Missing `bind=True` | Without `bind=True` the task function has no `self` reference, making `self.retry()` unavailable. All durable tasks must use `bind=True`. |
| Long mutex TTL | Setting a 24-hour TTL on a processing mutex means a crashed worker blocks all retries for 24 hours. Use short TTLs (e.g. `max_task_runtime + 20%`) for the mutex and long TTLs (24 h) only for completion checkpoints. |
| `time.sleep()` inside a task | Blocking a worker thread with `time.sleep()` ties up the worker for the sleep duration. Use `raise self.retry(countdown=N)` to defer with a delay instead. |
| `worker_prefetch_multiplier > 1` | Prefetching multiple messages per worker thread means a worker can hold messages it hasn't started yet. If it dies, those messages are invisible until the visibility timeout fires. Keep this at `1` for durable workloads. |
