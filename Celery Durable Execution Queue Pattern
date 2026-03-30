# Celery Durable Execution Queue Pattern

A general-purpose guide for building durable, idempotent Celery task pipelines that
survive worker crashes, pod/container deaths, and network partitions — without losing
work or producing duplicate side effects.

Compatible with **Redis / Valkey** as broker + backend and deployable on
**Kubernetes (EKS/GKE/AKS)** or **Amazon ECS (Fargate / EC2)**.

---

## Table of Contents

1. [Core Concepts](#1-core-concepts)
2. [Broker + Worker Configuration](#2-broker--worker-configuration)
3. [The Four-Layer Durable Execution Pattern](#3-the-four-layer-durable-execution-pattern)
   - [Layer 1 — Idempotency Pre-check](#layer-1--idempotency-pre-check)
   - [Layer 2 — Distributed Critical Section Lock (Mutex)](#layer-2--distributed-critical-section-lock-mutex)
   - [Layer 3 — Completion Checkpoints](#layer-3--completion-checkpoints)
   - [Layer 4 — Retry Instead of Ignore](#layer-4--retry-instead-of-ignore)
4. [Distributed Lock Implementation](#4-distributed-lock-implementation)
5. [Task Skeleton — Full Pattern](#5-task-skeleton--full-pattern)
6. [Chain / Chord Durability](#6-chain--chord-durability)
7. [Kubernetes-Specific Settings](#7-kubernetes-specific-settings)
8. [ECS-Specific Settings](#8-ecs-specific-settings)
9. [Operational Checklist](#9-operational-checklist)
10. [Anti-Patterns to Avoid](#10-anti-patterns-to-avoid)

---

## 1. Core Concepts

### What "durable execution" means

A task queue is **durable** when every message is processed **exactly once** even
when:

- A worker process is killed mid-task (OOM, SIGKILL, pod eviction)
- A network partition orphans the broker connection
- The same message is re-delivered because a visibility/acknowledgement timeout fires
- Multiple workers race to pick up the same message

Celery provides the plumbing (late acks, reject-on-worker-lost, retries).  You must
add application-level idempotency and distributed locking on top.

### The three failure modes you must design against

| Failure | Consequence without pattern | Consequence with pattern |
|---|---|---|
| Worker dies before task ACK | Message re-queued; new worker starts from scratch — may double-execute side effects | Completion checkpoint blocks duplicate; new worker detects and returns early |
| Two workers pick same message | Race condition; both execute non-idempotent operations | Critical-section lock lets only one proceed; contender retries |
| Worker completes task but ACK lost | Broker re-delivers; task runs again | Same as died-before-ACK above |

---

## 2. Broker + Worker Configuration

Apply these globally in `celery_app.py` before registering any tasks.

### Required settings

```python
# --- Durability core ---
app.conf.task_acks_late = True                         # ACK AFTER task completes, not before
app.conf.task_reject_on_worker_lost = True             # Re-queue on worker death (SIGKILL, OOM)
app.conf.task_acks_on_failure_or_timeout = True        # ACK permanent failures so they don't loop
app.conf.worker_prefetch_multiplier = 1                # One in-flight message per worker thread
app.conf.worker_cancel_long_running_tasks_on_connection_loss = False  # Don't kill running tasks

# --- Retry / backoff ---
app.conf.task_retry_backoff = True                     # Exponential backoff on retries
app.conf.task_retry_backoff_max = 600                  # Cap backoff at 10 minutes
app.conf.task_retry_jitter = True                      # Jitter prevents thundering herd

# --- Result expiry (prevent broker accumulation) ---
app.conf.result_expires = 3600                         # Expire results after 1 hour
app.conf.task_ignore_result = False                    # Keep results; set per-task if unneeded

# --- Broker transport (Redis / Valkey) ---
app.conf.broker_transport_options = {
    "visibility_timeout": 900,       # 15 min — must be > your longest task runtime
    "fanout_prefix": True,
    "fanout_patterns": True,
    "retry_on_timeout": True,
    "max_connections": 10,
}

# --- Result backend ---
app.conf.result_backend_transport_options = {
    "visibility_timeout": 900,
    "retry_on_timeout": True,
}
```

### Why `visibility_timeout` matters

Redis/Valkey implements "at-least-once" delivery via a visibility timeout.  A message
stays invisible to other workers while one worker holds it.  If the worker dies, the
message reappears after `visibility_timeout` seconds.

**Rule:** `visibility_timeout` **must be longer** than the maximum expected task
runtime.  If a task legitimately takes 12 minutes and the timeout is 10 minutes, the
broker re-delivers the live task to a second worker — both run simultaneously.

```
visibility_timeout > max_task_wall_clock_seconds + generous_buffer
```

---

## 3. The Four-Layer Durable Execution Pattern

```
┌─────────────────────────────────────────────────────┐
│  Task Entry                                         │
│                                                     │
│  1. Idempotency pre-check  ──► already done? ACK   │
│         │                                           │
│  2. Critical-section lock ──► contention? RETRY    │
│         │                                           │
│  3. Do non-idempotent work                         │
│         │                                           │
│  4. Set completion checkpoint                       │
│         │                                           │
│  Return / ACK                                       │
└─────────────────────────────────────────────────────┘
```

### Layer 1 — Idempotency Pre-check

Check a **persistent completion marker** (stored in Redis/Valkey or a database)
*before* acquiring any lock.  If the work is already done, return immediately with
a success result.  This short-circuits re-delivered tasks with zero contention.

```python
# Pseudo-code
if completion_checkpoint_exists(task_id):
    return {"status": "ALREADY_COMPLETED"}
```

### Layer 2 — Distributed Critical Section Lock (Mutex)

Use a short-TTL distributed lock to guarantee only one worker executes the
critical (non-idempotent) section at a time.

- **Fail fast** on acquire (`acquire_timeout=1s`) — do not block the worker
- On failure to acquire → **`self.retry(countdown=backoff)`**, never return/ignore
- Keep lock TTL longer than the operation it guards but shorter than 24 hours

```python
acquired = redis.set(lock_key, worker_id, nx=True, ex=lock_ttl_seconds)
if not acquired:
    raise self.retry(countdown=backoff)   # stays in queue
```

### Layer 3 — Completion Checkpoints

After each irreversible step, write a **persistent checkpoint key** with a long TTL
(must outlive any realistic re-delivery window, typically 30 min – 24 h).  The
checkpoint key is **never deleted on success** — its presence is the signal.

```
Step 1 completes → set "job:{id}:step1_done"  (TTL 24h)
Step 2 completes → set "job:{id}:step2_done"  (TTL 24h)
Finalization    → set "job:{id}:workflow_end" (TTL 24h)
```

On any re-entry, check checkpoints in order and skip completed steps.

### Layer 4 — Retry Instead of Ignore

**Never raise `Ignore` for recoverable failures.**  `Ignore` acknowledges the
message — the work is gone forever.  Raise `self.retry()` instead.

```python
# ❌ Wrong — message permanently lost
raise Ignore()

# ✅ Correct — message stays in queue; tried again later
raise self.retry(countdown=60)
```

Use `Ignore` only for true no-ops where you have already confirmed (via a checkpoint)
that the work completed.

---

## 4. Distributed Lock Implementation

A minimal, portable lock helper using Redis/Valkey:

```python
import uuid
import redis


class DistributedLock:
    """Simple distributed lock using Redis SET NX EX (Redlock-lite for single node)."""

    def __init__(self, client: redis.Redis):
        self.client = client

    def acquire(self, key: str, ttl: int, timeout: float = 1.0) -> str | None:
        """
        Try to acquire the lock.

        Returns the lock token (str) if acquired, None if the key is already held.
        Store the token; you must pass it to release() to prevent releasing another
        worker's lock.
        """
        token = str(uuid.uuid4())
        deadline = time.monotonic() + timeout
        while time.monotonic() < deadline:
            ok = self.client.set(key, token, nx=True, ex=ttl)
            if ok:
                return token
            time.sleep(0.05)
        return None

    def release(self, key: str, token: str) -> bool:
        """
        Release the lock only if we still own it (compare-and-delete via Lua).
        Returns True if released, False if already expired or taken by another worker.
        """
        lua = """
        if redis.call("get", KEYS[1]) == ARGV[1] then
            return redis.call("del", KEYS[1])
        else
            return 0
        end
        """
        result = self.client.eval(lua, 1, key, token)
        return bool(result)

    def is_set(self, key: str) -> bool:
        """Check whether a key (checkpoint) exists without acquiring."""
        return bool(self.client.exists(key))
```

> **Production note:** For multi-node Redis clusters use the
> [Redlock](https://redis.io/docs/latest/develop/use/patterns/distributed-locks/)
> algorithm or a library like `pottery` / `redis-py-lock`.

### Lock TTL guide

| Purpose | Suggested TTL | Rationale |
|---|---|---|
| Critical section (mutex) | `max_task_runtime + 20%` | Freed on success; TTL is the auto-release safety net |
| Post-step checkpoint | `24h` | Must outlive any re-delivery window |
| Workflow end checkpoint | `24h` | Same |
| Per-image / per-item lock | `max_item_runtime + 20%` | Freed on failure; kept on success |

---

## 5. Task Skeleton — Full Pattern

```python
from celery import Task
from celery.exceptions import Ignore
from requests.exceptions import ConnectionError, Timeout
from valkey.exceptions import ValkeyError   # or redis.exceptions.RedisError

@app.task(
    bind=True,
    base=YourBaseTask,
    name="tasks.my_durable_task",
    queue="my_queue",
    # ── Automatic retries for transient errors ──────────────────────────
    autoretry_for=(
        ConnectionError,
        Timeout,
        ValkeyError,
        OSError,
    ),
    retry_kwargs={"max_retries": 5},
    retry_backoff=True,
    retry_backoff_max=300,
    retry_jitter=True,
    # ── Durability settings (override per task if needed) ───────────────
    acks_late=True,
    reject_on_worker_lost=True,
)
def my_durable_task(self: Task, payload: dict) -> dict:
    job_id = payload["job_id"]
    lock = DistributedLock(redis_client)

    # ── Layer 1: Idempotency pre-check ─────────────────────────────────
    if lock.is_set(f"job:{job_id}:complete"):
        logger.info("Already completed — returning early", job_id=job_id)
        return {"status": "ALREADY_COMPLETED", "job_id": job_id}

    # ── Layer 2: Critical section lock ─────────────────────────────────
    lock_key = f"job:{job_id}:processing"
    token = lock.acquire(lock_key, ttl=900, timeout=1.0)

    if token is None:
        # Another worker is active. Retry instead of ignoring so the work
        # is not lost if the other worker dies.
        backoff = min(30 + self.request.retries * 15, 300)
        logger.warning(
            "Lock contention — retrying", job_id=job_id, backoff=backoff
        )
        raise self.retry(countdown=backoff)

    success = False
    try:
        # ── Layer 3: Check per-step checkpoints ────────────────────────
        step1_done = lock.is_set(f"job:{job_id}:step1_done")

        if not step1_done:
            _do_step_1(payload)                         # non-idempotent work
            redis_client.set(
                f"job:{job_id}:step1_done", "1", ex=86400   # 24h checkpoint
            )

        _do_step_2(payload)                             # idempotent; no checkpoint needed

        # ── Set final completion checkpoint ────────────────────────────
        redis_client.set(f"job:{job_id}:complete", "1", ex=86400)
        success = True
        return {"status": "DONE", "job_id": job_id}

    except Exception as exc:
        # Retryable errors re-raise here; autoretry_for handles them.
        # Permanent failures (ValueError, ValidationError, etc.) bubble up
        # and are ACK'd by task_acks_on_failure_or_timeout.
        raise

    finally:
        # ── Release the mutex only on failure ──────────────────────────
        # On success the task returns and Celery ACKs it; the critical-section
        # TTL will expire naturally and is short enough to cause no harm.
        # On failure we release immediately so the next retry can acquire.
        if not success:
            lock.release(lock_key, token)
```

---

## 6. Chain / Chord Durability

Celery `chain` and `chord` primitives need extra care.

### The chord callback problem

When a chord worker dies the callback may never fire.  Mitigate by:

1. Setting a generous `result_chord_join_timeout`:
   ```python
   app.conf.result_chord_join_timeout = 600   # 10 minutes
   ```

2. Using **checkpoint locks on the chord-initiating task** so a re-delivered
   initiator detects the chord was already dispatched and returns early instead
   of dispatching a duplicate chord.

   ```python
   chord_dispatched_key = f"job:{job_id}:chord_dispatched"
   if lock.is_set(chord_dispatched_key):
       return {"status": "CHORD_ALREADY_DISPATCHED"}

   # Acquire lock, build chord, dispatch, then set checkpoint
   token = lock.acquire(chord_dispatched_key, ttl=3600, timeout=1.0)
   if token:
       chord(header=tasks)(callback)           # dispatch
       redis_client.set(chord_dispatched_key, "1", ex=86400)
   ```

3. Each **body task** in the chord applies the full four-layer pattern
   independently — a failed OCR task, for example, should not bring down the
   whole chord.

### Chain link idempotency

Each link in a chain receives the return value of the previous link.  If a link
retries, it receives the same input again.  Design every link to be re-entrant:
check checkpoints before executing, then set them on completion.

---

## 7. Kubernetes-Specific Settings

### Graceful shutdown (the most critical Kubernetes concern)

Kubernetes sends **SIGTERM** before killing a pod.  Workers must catch SIGTERM and
stop accepting new tasks while finishing in-flight tasks.

```python
# In celery_app.py
app.conf.worker_shutdown_timeout = 60       # Max seconds to wait on SIGTERM before SIGKILL
```

Match your Kubernetes `terminationGracePeriodSeconds`:

```yaml
# deployment.yaml
spec:
  template:
    spec:
      terminationGracePeriodSeconds: 90     # Must be > worker_shutdown_timeout
      containers:
        - name: worker
          lifecycle:
            preStop:
              exec:
                # Give the worker a chance to finish current tasks
                command: ["/bin/sh", "-c", "celery -A app.celery.celery_app inspect active | sleep 5"]
```

### Resource limits prevent OOM cascades

```yaml
resources:
  requests:
    memory: "512Mi"
    cpu: "250m"
  limits:
    memory: "1Gi"        # OOM kills trigger task_reject_on_worker_lost — tasks re-queue
    cpu: "1000m"
```

### Liveness + readiness probes

```yaml
livenessProbe:
  exec:
    command: ["celery", "-A", "app.celery.celery_app", "inspect", "ping", "-d", "celery@$HOSTNAME"]
  initialDelaySeconds: 30
  periodSeconds: 60
  timeoutSeconds: 10
  failureThreshold: 3

readinessProbe:
  exec:
    command: ["celery", "-A", "app.celery.celery_app", "inspect", "ping", "-d", "celery@$HOSTNAME"]
  initialDelaySeconds: 10
  periodSeconds: 30
  timeoutSeconds: 5
```

### Horizontal Pod Autoscaler (HPA) considerations

HPA scales pods in and out based on CPU/memory.  When scaling **in**, pods receive
SIGTERM.  Because of `task_acks_late=True` + `task_reject_on_worker_lost=True`,
in-flight tasks are re-queued automatically — no special handling needed.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: External
      external:
        metric:
          name: celery_queue_length     # Custom metric via kube-state-metrics + KEDA
        target:
          type: AverageValue
          averageValue: "10"            # Scale up when average queue depth > 10
```

> Consider **KEDA** (Kubernetes Event-Driven Autoscaling) for queue-depth–based
> scaling using the Redis scaler.

### Anti-affinity for resilience

Spread workers across failure domains:

```yaml
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: celery-worker
          topologyKey: kubernetes.io/hostname
```

### `worker_max_tasks_per_child`

Restart worker child processes periodically to prevent slow memory leaks:

```python
app.conf.worker_max_tasks_per_child = 100    # recycle after 100 tasks
```

Set `terminationGracePeriodSeconds` high enough that in-progress tasks finish
before the recycled child's parent pod is replaced.

---

## 8. ECS-Specific Settings

### Graceful shutdown on ECS (SIGTERM → SIGKILL gap)

ECS sends SIGTERM and waits `stopTimeout` before SIGKILL.  Default is 30 s — too
short for long-running tasks.  Increase in your task definition:

```json
{
  "stopTimeout": 120
}
```

Set `worker_shutdown_timeout` to match:

```python
app.conf.worker_shutdown_timeout = 110   # 10s buffer before SIGKILL
```

### ECS task definition — resource + ulimit

```json
{
  "family": "celery-worker",
  "cpu": "1024",
  "memory": "2048",
  "containerDefinitions": [
    {
      "name": "worker",
      "command": ["celery", "-A", "app.celery.celery_app", "worker",
                  "--loglevel=info", "--concurrency=4", "-Q", "my_queue"],
      "stopTimeout": 120,
      "ulimits": [
        { "name": "nofile", "softLimit": 65536, "hardLimit": 65536 }
      ],
      "healthCheck": {
        "command": ["CMD", "celery", "-A", "app.celery.celery_app",
                    "inspect", "ping", "-d", "celery@$HOSTNAME"],
        "interval": 60,
        "timeout": 10,
        "retries": 3,
        "startPeriod": 30
      }
    }
  ]
}
```

### ECS Service Auto Scaling

Scale based on a custom CloudWatch metric published by your worker (queue depth):

```python
# Publish queue depth metric from a monitoring sidecar or Lambda
cloudwatch.put_metric_data(
    Namespace="Celery",
    MetricData=[{
        "MetricName": "QueueDepth",
        "Value": redis.llen("celery"),
        "Unit": "Count",
        "Dimensions": [{"Name": "QueueName", "Value": "my_queue"}],
    }]
)
```

```json
{
  "TargetTrackingScalingPolicyConfiguration": {
    "CustomizedMetricSpecification": {
      "MetricName": "QueueDepth",
      "Namespace": "Celery",
      "Statistic": "Average"
    },
    "TargetValue": 10.0,
    "ScaleInCooldown": 120,
    "ScaleOutCooldown": 30
  }
}
```

### Fargate vs EC2 launch type

| | Fargate | EC2 |
|---|---|---|
| SIGTERM behavior | Same as above | Same as above |
| `worker_max_tasks_per_child` | Recommended — limits memory growth | Recommended |
| Host process isolation | Full container isolation | Shared kernel |
| Cost profile | Higher per vCPU; simpler ops | Lower at scale; more ops overhead |

---

## 9. Operational Checklist

Before deploying a new durable task to production:

### Broker + worker
- [ ] `task_acks_late = True`
- [ ] `task_reject_on_worker_lost = True`
- [ ] `task_acks_on_failure_or_timeout = True`
- [ ] `worker_prefetch_multiplier = 1`
- [ ] `visibility_timeout > max_task_runtime`

### Task code
- [ ] Idempotency pre-check at entry (reads completion checkpoint)
- [ ] Distributed lock acquired before any non-idempotent work
- [ ] Contention path raises `self.retry()` — **never** `Ignore` or `return`
- [ ] Completion checkpoint written after every irreversible step
- [ ] Critical-section lock released only on failure (kept on success OR
      released and checkpoint written)
- [ ] `autoretry_for` covers all transient exception types
- [ ] `max_retries` set (prevent infinite retry loops)
- [ ] Valkey/Redis connections closed in `finally` blocks

### Kubernetes
- [ ] `terminationGracePeriodSeconds > worker_shutdown_timeout`
- [ ] `worker_shutdown_timeout` set in Celery config
- [ ] Memory limits set (OOM re-queues via `reject_on_worker_lost`)
- [ ] Liveness probe configured

### ECS
- [ ] `stopTimeout >= worker_shutdown_timeout + 10s`
- [ ] Health check configured in task definition
- [ ] Auto scaling based on queue depth metric

---

## 10. Anti-Patterns to Avoid

### `raise Ignore()` for "another worker is handling this"

```python
# ❌ — message is ACK'd and permanently lost if holding worker dies
if lock_exists:
    raise Ignore()

# ✅ — message stays in queue until the work is confirmed done
if lock_exists:
    raise self.retry(countdown=30)
```

### Long blanket lock TTLs (e.g. 24 hours on a mutex)

```python
# ❌ — worker dies mid-task; lock blocks all retries for 24 hours
redis.set("job:123:lock", 1, ex=86400)

# ✅ — short mutex TTL; separate long-lived checkpoint key
redis.set("job:123:processing", 1, ex=900)       # mutex — 15 min
# after work done:
redis.set("job:123:complete", 1, ex=86400)        # checkpoint — 24 h
```

### Forgetting `bind=True` prevents `self.retry()`

```python
# ❌ — no access to self; cannot call self.retry()
@app.task(autoretry_for=(Timeout,))
def my_task(payload):
    ...

# ✅
@app.task(bind=True, autoretry_for=(Timeout,))
def my_task(self, payload):
    ...
```

### Using `time.sleep()` for lock-contention waiting

```python
# ❌ — blocks the worker thread; no other tasks can run
time.sleep(60)
raise self.retry()

# ✅ — releases the thread back to the pool during the wait
raise self.retry(countdown=60)
```

### No connection cleanup in `finally`

```python
# ❌ — Redis connections accumulate; pool exhausts under load
def my_task(self, payload):
    r = redis.Redis(...)
    r.get("key")
    # no cleanup

# ✅
def my_task(self, payload):
    r = redis.Redis(...)
    try:
        r.get("key")
    finally:
        r.close()
```

### Ignoring `visibility_timeout` in Celery config

Not setting `broker_transport_options["visibility_timeout"]` leaves it at the
Redis default (300 s).  Any task running longer than 5 minutes will be
re-delivered to a second worker while the first is still running.

---

## Quick Reference — Lock Types and Lifetimes

```
Key naming convention:
  {domain}:{entity_id}:lock:{lock_type}

Examples:
  job:abc123:lock:processing      → critical-section mutex   (TTL = task runtime + buffer)
  job:abc123:lock:step1_complete  → step checkpoint          (TTL = 24h)
  job:abc123:lock:workflow_end    → completion checkpoint    (TTL = 24h)
  job:abc123:lock:item:img01      → per-item OCR/processor   (TTL = item runtime + buffer)

Release rules:
  Mutex       → release on failure; keep (let expire) on success
  Checkpoint  → never release; long TTL; presence = done
```
