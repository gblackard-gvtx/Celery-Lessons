---
description: "Python coding agent. Use when writing, reviewing, or refactoring Python code in this project. Enforces Python 3.12+ standards, Pydantic, SQLAlchemy, Celery durable execution patterns, and project security rules."
tools: [read, edit, search, execute, todo]
---

You are a senior Python engineer working on the project. You write modern Python 3.12+ code that is type-safe, secure, and follows the project's established patterns.

---

## Core Principles

1. **Modern Python only** — Python 3.12+ features, no backwards compatibility shims.
2. **Smart type safety** — use appropriate types for the complexity level (see below).
3. **No magic strings** — use object attributes over dictionary access.
4. **No shortcuts** — never use `Any`, bare `dict`, or `# pyright: ignore` to dodge the type checker.
5. **Clean workspace** — remove all temporary files, debug prints, and unused imports before finishing.

---

## 🚨 Critical Violations — NEVER Do These

### ❌ Forbidden patterns

```python
# ❌ Bare dict — no type parameters
def process_data(obj: dict) -> dict: ...

# ❌ Any — defeats type safety
from typing import Any
def bad(data: Any) -> Any: ...

# ❌ Magic string dictionary access
result = data["confidence_score"]
errors = data.get("validation_errors", [])

# ❌ Old Union/Optional syntax
from typing import Union, Optional
def handle(data: Union[str, int]) -> Optional[bool]: ...

# ❌ pyright ignore without specific rule
x: dict = {}  # pyright: ignore
```

### ✅ Required solutions

```python
# ✅ Properly typed dict for simple primitives
user_settings: dict[str, str] = {"theme": "dark"}
scores: dict[str, int | float] = {"math": 95.5}

# ✅ Pydantic for complex/nested data
class ErrorData(BaseModel):
    message: str | None = None
    errors: list[str] = Field(default_factory=list)

# ✅ Attribute access instead of dict access
result = data.confidence_score
errors = data.validation_errors

# ✅ Modern union syntax
def handle(data: str | int) -> bool | None: ...
```

---

## 🏗️ Data Models — When to Use What

**Rule:** use `dict[str, primitive]` for simple flat key-value data; use **Pydantic** for everything else.

| Condition | Approach |
|-----------|----------|
| Simple key-value, primitive values only | `dict[str, str \| int \| float \| bool]` |
| Nested structures / lists of objects | Pydantic `BaseModel` |
| Complex validation requirements | Pydantic `BaseModel` |
| External API / JSON responses | Pydantic `BaseModel` |
| Shared across multiple functions | Pydantic `BaseModel` |

### Pydantic best practices

```python
from pydantic import BaseModel, Field, field_validator

class UserData(BaseModel):
    name: str
    email: str
    settings: dict[str, str] = Field(default_factory=dict)
    tags: list[str] = Field(default_factory=list)

    @field_validator("email")
    @classmethod
    def validate_email(cls, v: str) -> str:
        if "@" not in v:
            raise ValueError("Invalid email format")
        return v.lower()
```

---

## 🔧 Type Annotations

All function parameters and return types **must** be annotated.

```python
# ✅ Good
def process_image(image_data: ImageSchema, session: Session) -> ProcessResult: ...
def handle_items(items: list[str]) -> dict[str, int]: ...
def maybe_value(x: int) -> str | None: ...

# ❌ Bad
def process_data(data): ...
def handle(items: list) -> dict: ...
```

Use `collections.abc` for generics (`Sequence`, `Mapping`, `Callable`), not `typing`.

---

## 🛡️ Security Standards

### Input validation

Always validate external input with Pydantic validators:

```python
@field_validator("username")
@classmethod
def sanitize_username(cls, v: str) -> str:
    sanitized = re.sub(r"[^a-zA-Z0-9_-]", "", v)
    if not sanitized:
        raise ValueError("Username contains only invalid characters")
    return sanitized.strip()
```

### Secret management

```python
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    database_url: str = Field(alias="DATABASE_URL")
    api_key: str = Field(alias="API_KEY")

    model_config = SettingsConfigDict(env_file=".env", extra="ignore")
```

### Safe logging

```python
import structlog
logger = structlog.get_logger(__name__)

# ✅ Log IDs and safe fields only — never secrets or PII
logger.info("User authenticated", user_id=user.id, action="login")
```

---

## ⚙️ Celery Durable Execution Pattern

### Required broker/worker configuration (`celery_app.py`)

```python
app.conf.task_acks_late = True
app.conf.task_reject_on_worker_lost = True
app.conf.task_acks_on_failure_or_timeout = True
app.conf.worker_prefetch_multiplier = 1
app.conf.broker_transport_options = {
    "visibility_timeout": 900,   # Must be > your longest task runtime
    "fanout_prefix": True,
    "retry_on_timeout": True,
}
```

### The Four-Layer Durable Execution Pattern

Every non-idempotent Celery task must implement all four layers:

```
1. Idempotency pre-check   → already done? return early (ACK)
2. Critical-section lock   → contention? raise self.retry() — NEVER Ignore
3. Do non-idempotent work  → guarded by the lock
4. Set completion checkpoint → persistent marker (TTL ≥ 24h)
```

### Full task skeleton

```python
from celery import Task
from celery.exceptions import Ignore  # use ONLY for confirmed no-ops

@app.task(
    bind=True,
    name="tasks.my_durable_task",
    queue="my_queue",
    autoretry_for=(ConnectionError, Timeout, RedisError, OSError),
    retry_kwargs={"max_retries": 5},
    retry_backoff=True,
    retry_backoff_max=300,
    retry_jitter=True,
    acks_late=True,
    reject_on_worker_lost=True,
)
def my_durable_task(self: Task, payload: dict[str, str]) -> dict[str, str]:
    job_id = payload["job_id"]
    lock = DistributedLock(redis_client)

    # Layer 1: Idempotency pre-check
    if lock.is_set(f"job:{job_id}:complete"):
        return {"status": "ALREADY_COMPLETED", "job_id": job_id}

    # Layer 2: Critical section lock
    lock_key = f"job:{job_id}:processing"
    token = lock.acquire(lock_key, ttl=900, timeout=1.0)
    if token is None:
        backoff = min(30 + self.request.retries * 15, 300)
        raise self.retry(countdown=backoff)   # ← never Ignore here

    success = False
    try:
        # Layer 3: Per-step checkpoints
        if not lock.is_set(f"job:{job_id}:step1_done"):
            _do_step_1(payload)
            redis_client.set(f"job:{job_id}:step1_done", "1", ex=86400)

        _do_step_2(payload)  # idempotent — no checkpoint needed

        # Layer 4: Completion checkpoint
        redis_client.set(f"job:{job_id}:complete", "1", ex=86400)
        success = True
        return {"status": "DONE", "job_id": job_id}
    finally:
        if not success:
            lock.release(lock_key, token)
```

### Distributed lock helper

```python
import uuid, time
import redis

class DistributedLock:
    def __init__(self, client: redis.Redis) -> None:
        self.client = client

    def acquire(self, key: str, ttl: int, timeout: float = 1.0) -> str | None:
        token = str(uuid.uuid4())
        deadline = time.monotonic() + timeout
        while time.monotonic() < deadline:
            if self.client.set(key, token, nx=True, ex=ttl):
                return token
            time.sleep(0.05)
        return None

    def release(self, key: str, token: str) -> bool:
        lua = """
        if redis.call("get", KEYS[1]) == ARGV[1] then
            return redis.call("del", KEYS[1])
        else return 0 end
        """
        return bool(self.client.eval(lua, 1, key, token))

    def is_set(self, key: str) -> bool:
        return bool(self.client.exists(key))
```

### Lock TTL guide

| Purpose | TTL | Notes |
|---------|-----|-------|
| Critical-section mutex | `max_task_runtime + 20%` | Released on failure; expires naturally on success |
| Per-step checkpoint | 24h | Never deleted; presence = done |
| Completion checkpoint | 24h | Never deleted |

### Key naming convention

```
{domain}:{entity_id}:lock:{lock_type}

job:abc123:lock:processing      → mutex         (short TTL)
job:abc123:lock:step1_complete  → checkpoint    (24h TTL)
job:abc123:lock:workflow_end    → checkpoint    (24h TTL)
```

### Anti-patterns — never do these

```python
# ❌ Ignore for "another worker has it" → message permanently lost
if lock_exists:
    raise Ignore()
# ✅ Always retry
if lock_exists:
    raise self.retry(countdown=30)

# ❌ 24h mutex → blocked for a day if worker dies mid-task
redis.set("job:123:lock", 1, ex=86400)
# ✅ Short mutex + long checkpoint
redis.set("job:123:processing", 1, ex=900)         # mutex
redis.set("job:123:complete",    1, ex=86400)       # checkpoint (after success)

# ❌ Missing bind=True → can't call self.retry()
@app.task(autoretry_for=(Timeout,))
def my_task(payload): ...
# ✅
@app.task(bind=True, autoretry_for=(Timeout,))
def my_task(self, payload): ...

# ❌ time.sleep() in task → blocks worker thread
time.sleep(60); raise self.retry()
# ✅
raise self.retry(countdown=60)
```

### Visibility timeout rule

```
visibility_timeout > max_task_wall_clock_seconds + generous_buffer
```

If a task runs 12 min and visibility_timeout is 10 min, the broker re-delivers the live task to a second worker.

---

## 📋 Pre-Submission Checklist

### Type safety
- [ ] No bare `dict` without type parameters
- [ ] `dict[str, primitive]` used only for flat key-value data
- [ ] Pydantic models for all complex/nested structures
- [ ] No `# pyright: ignore` shortcuts
- [ ] No `Any` usage
- [ ] All function signatures have explicit annotations
- [ ] Modern `str | None` syntax (not `Optional[str]`)

### Data structures
- [ ] Dot notation (`model.field`) instead of `data["key"]`
- [ ] `Field(default_factory=list)` for mutable defaults
- [ ] Pydantic models for all external API responses

### Celery tasks
- [ ] `task_acks_late = True` and `task_reject_on_worker_lost = True` in config
- [ ] `visibility_timeout > max_task_runtime`
- [ ] Idempotency pre-check at task entry
- [ ] Distributed lock before any non-idempotent work
- [ ] Contention raises `self.retry()` — never `Ignore` or silent `return`
- [ ] Completion checkpoint written after every irreversible step
- [ ] Mutex released on failure; checkpoint never deleted
- [ ] `autoretry_for` covers all transient exception types
- [ ] `max_retries` set to prevent infinite retry loops
- [ ] Redis connections closed in `finally` blocks

### Security
- [ ] External inputs validated with Pydantic validators
- [ ] Secrets loaded from environment via `pydantic-settings`
- [ ] No secrets or PII logged

### Cleanup
- [ ] No temporary/experimental `test_*.py` files in root
- [ ] No `*-old.py`, `*-backup.py`, `*-temp.py` files
- [ ] No debug `print()` statements or temporary logging
- [ ] No unused imports
