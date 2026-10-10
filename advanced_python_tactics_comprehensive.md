# Advanced Python Development Tactics
### Comprehensive Production-Ready Guide with Full Examples

This guide covers advanced Python development tactics beyond core language features, OOP, and basic concurrency. It focuses on production-grade patterns, performance, resilience, observability, and maintainability for real-world systems.

**Target Python**: 3.11+  
**Style**: Type-hinted, explicit, production-oriented

---

## Table of Contents

1. [Advanced Concurrency & Parallelism](#1-advanced-concurrency--parallelism)
2. [Performance Engineering](#2-performance-engineering)
3. [Advanced Memory Management](#3-advanced-memory-management)
4. [Functional-Style Techniques](#4-functional-style-techniques)
5. [Metaprogramming & Runtime Code Generation](#5-metaprogramming--runtime-code-generation)
6. [Robust Error Handling & Resilience](#6-robust-error-handling--resilience)
7. [Observability & Production Hardening](#7-observability--production-hardening)
8. [Configuration & Secrets Management](#8-configuration--secrets-management)
9. [Advanced Validation & Data Modeling](#9-advanced-validation--data-modeling)
10. [Caching Strategies](#10-caching-strategies)
11. [API & Library Design](#11-api--library-design)
12. [Testing at a Higher Level](#12-testing-at-a-higher-level)
13. [Packaging & Distribution Advanced](#13-packaging--distribution-advanced)
14. [Security Hardening](#14-security-hardening)
15. [Interoperability & Foreign Function Interfaces](#15-interoperability--foreign-function-interfaces)
16. [Advanced Async Patterns (Queues, Workers, Backpressure)](#16-advanced-async-patterns-queues-workers-backpressure)
17. [Idempotency, Outbox & Distributed Patterns](#17-idempotency-outbox--distributed-patterns)
18. [Advanced Typing Techniques](#18-advanced-typing-techniques)
19. [Domain-Driven Design Tactics in Python](#19-domain-driven-design-tactics-in-python)
20. [Rate Limiting & Bulkheads](#20-rate-limiting--bulkheads)
21. [Lazy Loading, Import Hooks & Plugin Discovery](#21-lazy-loading-import-hooks--plugin-discovery)
22. [Buffer Protocol, Memoryview & Zero-Copy](#22-buffer-protocol-memoryview--zero-copy)

---

## 1. Advanced Concurrency & Parallelism

### 1.1 Structured Concurrency with TaskGroup (Python 3.11+)

```python
import asyncio
import logging
from typing import Any

logger = logging.getLogger(__name__)


async def fetch_user(user_id: int) -> dict[str, Any]:
    await asyncio.sleep(0.1)
    return {"id": user_id, "name": f"User {user_id}"}


async def fetch_orders(user_id: int) -> list[dict]:
    await asyncio.sleep(0.15)
    return [{"id": 1, "user_id": user_id, "total": 99.9}]


async def fetch_preferences(user_id: int) -> dict:
    await asyncio.sleep(0.05)
    return {"theme": "dark", "notifications": True}


async def load_dashboard(user_id: int) -> dict[str, Any]:
    """
    Structured concurrency: if any task fails, all others are cancelled.
    Much safer than asyncio.gather() + manual cancellation.
    """
    async with asyncio.TaskGroup() as tg:
        user_task = tg.create_task(fetch_user(user_id), name="fetch_user")
        orders_task = tg.create_task(fetch_orders(user_id), name="fetch_orders")
        prefs_task = tg.create_task(fetch_preferences(user_id), name="fetch_prefs")

    return {
        "user": user_task.result(),
        "orders": orders_task.result(),
        "preferences": prefs_task.result(),
    }


async def load_dashboard_with_timeout(user_id: int, timeout: float = 2.0) -> dict:
    try:
        async with asyncio.timeout(timeout):
            return await load_dashboard(user_id)
    except TimeoutError:
        logger.error("Dashboard load timed out for user %s", user_id)
        raise
```

### 1.2 Hybrid: asyncio + ProcessPoolExecutor for CPU-bound work

```python
import asyncio
from concurrent.futures import ProcessPoolExecutor
from functools import partial


def heavy_computation(data: list[float]) -> float:
    # CPU-intensive work
    return sum(x ** 2 for x in data)


async def run_cpu_bound(data: list[float]) -> float:
    loop = asyncio.get_running_loop()
    with ProcessPoolExecutor(max_workers=4) as pool:
        result = await loop.run_in_executor(
            pool,
            partial(heavy_computation, data)
        )
    return result
```

### 1.3 Decision Framework

| Workload Type       | Recommended Tool              | Notes                              |
|---------------------|-------------------------------|------------------------------------|
| I/O-bound           | `asyncio` + TaskGroup         | Preferred for modern services      |
| CPU-bound           | `ProcessPoolExecutor`         | Bypasses GIL                       |
| Mixed               | asyncio + run_in_executor     | Common in web services             |
| High concurrency    | asyncio + uvloop (optional)   | Performance boost on Linux         |
| Simple scripts      | `concurrent.futures`          | Easier than full asyncio           |

---

## 2. Performance Engineering

### 2.1 Profiling Workflow

```bash
# Quick overview
python -m cProfile -o output.prof your_script.py
snakeviz output.prof

# Production sampling (no code changes)
py-spy record -o profile.svg --pid <PID>
py-spy top --pid <PID>

# CPU + memory + GPU
scalene your_script.py
```

### 2.2 Targeted Optimization Example

```python
from functools import lru_cache
from numba import njit


@lru_cache(maxsize=None)
def fibonacci_cached(n: int) -> int:
    if n < 2:
        return n
    return fibonacci_cached(n - 1) + fibonacci_cached(n - 2)


@njit
def fibonacci_numba(n: int) -> int:
    a, b = 0, 1
    for _ in range(n):
        a, b = b, a + b
    return a
```

### 2.3 Production Performance Checklist

- Measure before optimizing (use production-like data)
- Prefer algorithmic improvements over micro-optimizations
- Use `slots=True` for high-volume objects
- Vectorize with NumPy when working with arrays
- Consider mypyc / Cython / Numba only for proven hotspots
- Cache aggressively but invalidate correctly

---

## 3. Advanced Memory Management

### 3.1 Weak References for Caches

```python
from weakref import WeakValueDictionary
from typing import Any


class ExpensiveObject:
    def __init__(self, key: str):
        self.key = key
        self.data = [0] * 1_000_000  # simulate heavy resource


_object_cache: WeakValueDictionary[str, ExpensiveObject] = WeakValueDictionary()


def get_expensive_object(key: str) -> ExpensiveObject:
    obj = _object_cache.get(key)
    if obj is None:
        obj = ExpensiveObject(key)
        _object_cache[key] = obj
    return obj
```

### 3.2 `__slots__` at Scale

```python
from dataclasses import dataclass


@dataclass(slots=True)
class Event:
    event_id: str
    timestamp: float
    payload: dict
```

### 3.3 Avoiding Common Memory Pitfalls

- Circular references → use `weakref` or break cycles
- Large lists of objects → prefer `__slots__` or generators
- Unbounded caches → always set maxsize / TTL / weakrefs
- `__del__` is unreliable → prefer context managers

---

## 4. Functional-Style Techniques

### 4.1 Pipeline Style

```python
from functools import reduce, partial
from itertools import islice
from typing import Iterable, Callable, Any


def pipe(data: Any, *functions: Callable) -> Any:
    return reduce(lambda acc, f: f(acc), functions, data)


def batch(iterable: Iterable, size: int):
    iterator = iter(iterable)
    while chunk := list(islice(iterator, size)):
        yield chunk
```

### 4.2 Result / Either Style

```python
from typing import Generic, TypeVar, Union, Callable
from dataclasses import dataclass

T = TypeVar("T")
E = TypeVar("E")


@dataclass(frozen=True, slots=True)
class Ok(Generic[T]):
    value: T

    def map(self, func: Callable[[T], Any]) -> "Ok | Err":
        return Ok(func(self.value))

    def is_ok(self) -> bool:
        return True


@dataclass(frozen=True, slots=True)
class Err(Generic[E]):
    error: E

    def map(self, func: Callable) -> "Err":
        return self

    def is_ok(self) -> bool:
        return False


Result = Union[Ok[T], Err[E]]
```

---

## 5. Metaprogramming & Runtime Code Generation

### 5.1 Safe Dynamic Class Creation

```python
from typing import Type, Any


def create_model(name: str, fields: dict[str, Type]) -> Type:
    def __init__(self, **kwargs):
        for key, value in kwargs.items():
            if key not in fields:
                raise TypeError(f"Unexpected field: {key}")
            setattr(self, key, value)

    def __repr__(self):
        attrs = ", ".join(f"{k}={getattr(self, k)!r}" for k in fields)
        return f"{name}({attrs})"

    namespace = {
        "__init__": __init__,
        "__repr__": __repr__,
        "__annotations__": fields,
    }
    return type(name, (), namespace)
```

**Rule**: Prefer dataclasses, Pydantic, or closures over `exec`/`eval`.

---

## 6. Robust Error Handling & Resilience

### 6.1 Custom Exception Hierarchy

```python
class AppError(Exception):
    def __init__(self, message: str, *, code: str | None = None):
        super().__init__(message)
        self.code = code or self.__class__.__name__


class NotFoundError(AppError):
    pass


class ValidationError(AppError):
    pass


class ExternalServiceError(AppError):
    def __init__(self, message: str, *, service: str, original: Exception | None = None):
        super().__init__(message, code="EXTERNAL_SERVICE_ERROR")
        self.service = service
        self.original = original
```

### 6.2 Circuit Breaker

```python
import time
from enum import Enum, auto
from typing import Callable, TypeVar
from functools import wraps

T = TypeVar("T")


class CircuitState(Enum):
    CLOSED = auto()
    OPEN = auto()
    HALF_OPEN = auto()


class CircuitBreaker:
    def __init__(
        self,
        failure_threshold: int = 5,
        recovery_timeout: float = 30.0,
        expected_exception: type[Exception] = Exception,
    ):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.expected_exception = expected_exception
        self.failure_count = 0
        self.last_failure_time: float | None = None
        self.state = CircuitState.CLOSED

    def __call__(self, func: Callable[..., T]) -> Callable[..., T]:
        @wraps(func)
        def wrapper(*args, **kwargs) -> T:
            if self.state == CircuitState.OPEN:
                if self._should_try_reset():
                    self.state = CircuitState.HALF_OPEN
                else:
                    raise ExternalServiceError(
                        "Circuit breaker is OPEN",
                        service=func.__name__,
                    )
            try:
                result = func(*args, **kwargs)
                self._on_success()
                return result
            except self.expected_exception:
                self._on_failure()
                raise
        return wrapper

    def _should_try_reset(self) -> bool:
        return (
            self.last_failure_time is not None
            and time.monotonic() - self.last_failure_time >= self.recovery_timeout
        )

    def _on_success(self) -> None:
        self.failure_count = 0
        self.state = CircuitState.CLOSED

    def _on_failure(self) -> None:
        self.failure_count += 1
        self.last_failure_time = time.monotonic()
        if self.failure_count >= self.failure_threshold:
            self.state = CircuitState.OPEN
```

---

## 7. Observability & Production Hardening

### 7.1 Structured Logging with structlog

```python
import structlog
import logging

structlog.configure(
    processors=[
        structlog.contextvars.merge_contextvars,
        structlog.processors.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.StackInfoRenderer(),
        structlog.processors.format_exc_info,
        structlog.processors.JSONRenderer(),
    ],
    wrapper_class=structlog.make_filtering_bound_logger(logging.INFO),
    context_class=dict,
    logger_factory=structlog.PrintLoggerFactory(),
    cache_logger_on_first_use=True,
)

logger = structlog.get_logger()
```

### 7.2 Context Variables for Request-Scoped Data

```python
from contextvars import ContextVar
from typing import Optional

request_id_var: ContextVar[Optional[str]] = ContextVar("request_id", default=None)
user_id_var: ContextVar[Optional[str]] = ContextVar("user_id", default=None)


def set_request_context(request_id: str, user_id: str | None = None) -> None:
    request_id_var.set(request_id)
    if user_id:
        user_id_var.set(user_id)
```

---

## 8. Configuration & Secrets Management

### 8.1 Pydantic Settings

```python
from pydantic_settings import BaseSettings, SettingsConfigDict
from pydantic import Field, SecretStr


class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        extra="ignore",
    )

    app_name: str = "MyApp"
    environment: str = Field(default="development", alias="APP_ENV")
    debug: bool = False
    database_url: SecretStr
    redis_url: str = "redis://localhost:6379/0"
    api_key: SecretStr
    max_connections: int = 100

    @property
    def is_production(self) -> bool:
        return self.environment == "production"
```

---

## 9. Advanced Validation & Data Modeling

### 9.1 Pydantic v2 Advanced Patterns

```python
from pydantic import BaseModel, Field, field_validator, model_validator, ConfigDict
from typing import Literal
from datetime import datetime


class OrderItem(BaseModel):
    product_id: str
    quantity: int = Field(gt=0, le=100)
    unit_price: float = Field(gt=0)


class Order(BaseModel):
    model_config = ConfigDict(str_strip_whitespace=True, frozen=True)

    order_id: str
    customer_id: str
    items: list[OrderItem]
    status: Literal["pending", "confirmed", "shipped", "cancelled"] = "pending"
    created_at: datetime = Field(default_factory=datetime.utcnow)

    @field_validator("items")
    @classmethod
    def must_have_items(cls, v: list[OrderItem]) -> list[OrderItem]:
        if not v:
            raise ValueError("Order must contain at least one item")
        return v

    @model_validator(mode="after")
    def check_total_items(self) -> "Order":
        if sum(item.quantity for item in self.items) > 500:
            raise ValueError("Total quantity exceeds limit")
        return self

    @property
    def total_amount(self) -> float:
        return sum(item.quantity * item.unit_price for item in self.items)
```

### 9.2 Discriminated Unions

```python
from typing import Annotated, Union, Literal
from pydantic import Field, BaseModel


class EmailNotification(BaseModel):
    type: Literal["email"] = "email"
    recipient: str
    subject: str


class SMSNotification(BaseModel):
    type: Literal["sms"] = "sms"
    phone: str
    message: str


Notification = Annotated[
    Union[EmailNotification, SMSNotification],
    Field(discriminator="type"),
]
```

---

## 10. Caching Strategies

### 10.1 TTL Cache

```python
import time
from typing import Any, Optional
import hashlib
import json


class TTLCache:
    def __init__(self, ttl_seconds: float = 300.0, maxsize: int = 1024):
        self.ttl = ttl_seconds
        self.maxsize = maxsize
        self._store: dict[str, tuple[float, Any]] = {}

    def get(self, key: str) -> Optional[Any]:
        item = self._store.get(key)
        if item is None:
            return None
        expires_at, value = item
        if time.monotonic() > expires_at:
            del self._store[key]
            return None
        return value

    def set(self, key: str, value: Any) -> None:
        if len(self._store) >= self.maxsize:
            oldest_key = next(iter(self._store))
            del self._store[oldest_key]
        self._store[key] = (time.monotonic() + self.ttl, value)
```

### 10.2 Production Caching Rules

- Define TTL or explicit invalidation
- Protect against cache stampede
- Use Redis/Memcached for multi-instance deployments
- Monitor hit rates

---

## 11. API & Library Design

### 11.1 Clean Public API

```python
# __init__.py
from .core import OrderService, create_order_service
from .models import Order, OrderItem, OrderStatus
from .exceptions import OrderError, NotFoundError, ValidationError

__all__ = [
    "OrderService",
    "create_order_service",
    "Order",
    "OrderItem",
    "OrderStatus",
    "OrderError",
    "NotFoundError",
    "ValidationError",
]
```

### 11.2 Deprecation Helper

```python
import warnings
from functools import wraps


def deprecated(message: str, version: str = ""):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            warnings.warn(
                f"{func.__name__} is deprecated"
                + (f" since version {version}" if version else "")
                + f". {message}",
                category=DeprecationWarning,
                stacklevel=2,
            )
            return func(*args, **kwargs)
        return wrapper
    return decorator
```

---

## 12. Testing at a Higher Level

### 12.1 Property-Based Testing

```python
from hypothesis import given, strategies as st


@given(st.lists(st.integers(), min_size=0, max_size=10_000))
def test_sort_properties(data: list[int]):
    sorted_data = sorted(data)
    assert sorted(sorted_data) == sorted_data
    assert len(sorted_data) == len(data)
    for i in range(len(sorted_data) - 1):
        assert sorted_data[i] <= sorted_data[i + 1]
```

### 12.2 Test Data Builders

```python
from dataclasses import dataclass, field
from datetime import datetime


@dataclass
class UserBuilder:
    user_id: str = "user-123"
    email: str = "test@example.com"
    full_name: str = "Test User"
    is_active: bool = True
    created_at: datetime = field(default_factory=datetime.utcnow)

    def with_email(self, email: str) -> "UserBuilder":
        self.email = email
        return self

    def inactive(self) -> "UserBuilder":
        self.is_active = False
        return self

    def build(self) -> "User":
        return User(
            user_id=self.user_id,
            email=self.email,
            full_name=self.full_name,
            is_active=self.is_active,
            created_at=self.created_at,
        )
```

---

## 13. Packaging & Distribution Advanced

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "my-advanced-lib"
version = "1.2.0"
description = "Production-grade library"
readme = "README.md"
requires-python = ">=3.11"
license = "MIT"
dependencies = [
    "pydantic>=2.5",
    "httpx>=0.27",
    "structlog>=24.1",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0",
    "pytest-asyncio>=0.23",
    "hypothesis>=6.90",
    "mypy>=1.8",
    "ruff>=0.3",
]

[project.scripts]
mycli = "my_advanced_lib.cli:main"

[tool.hatch.build.targets.wheel]
packages = ["src/my_advanced_lib"]

[tool.mypy]
python_version = "3.11"
strict = true
```

---

## 14. Security Hardening

- Never use `pickle` or `yaml.unsafe_load` on untrusted data
- Always validate external input (preferably with Pydantic)
- Use parameterized queries only
- Store secrets in a secret manager
- Keep dependencies updated and scanned
- Run with least privilege

```python
import json
from pydantic import BaseModel


def load_order(data: str) -> "Order":
    raw = json.loads(data)
    return Order.model_validate(raw)  # validation happens here
```

---

## 15. Interoperability & Foreign Function Interfaces

Prefer **cffi** or **PyO3** (Rust) over raw `ctypes` for new code.

Use native extensions only for proven hotspots or when integrating existing high-performance libraries.

---

## 16. Advanced Async Patterns (Queues, Workers, Backpressure)

### 16.1 Worker Pool with asyncio.Queue + Backpressure

```python
import asyncio
import logging
from typing import Any, Callable, Awaitable

logger = logging.getLogger(__name__)


async def worker(
    name: str,
    queue: asyncio.Queue,
    handler: Callable[[Any], Awaitable[None]],
) -> None:
    while True:
        item = await queue.get()
        try:
            await handler(item)
        except Exception:
            logger.exception("Worker %s failed processing item", name)
        finally:
            queue.task_done()


async def run_worker_pool(
    items: list[Any],
    handler: Callable[[Any], Awaitable[None]],
    num_workers: int = 5,
    max_queue_size: int = 100,
) -> None:
    queue: asyncio.Queue = asyncio.Queue(maxsize=max_queue_size)

    workers = [
        asyncio.create_task(worker(f"worker-{i}", queue, handler))
        for i in range(num_workers)
    ]

    for item in items:
        await queue.put(item)  # this will wait if queue is full → backpressure

    await queue.join()

    for w in workers:
        w.cancel()
    await asyncio.gather(*workers, return_exceptions=True)
```

### 16.2 Semaphore for Concurrency Limiting

```python
async def bounded_gather(
    coros: list[Awaitable[Any]],
    limit: int = 10,
) -> list[Any]:
    semaphore = asyncio.Semaphore(limit)

    async def sem_task(coro: Awaitable[Any]) -> Any:
        async with semaphore:
            return await coro

    return await asyncio.gather(*(sem_task(c) for c in coros))
```

---

## 17. Idempotency, Outbox & Distributed Patterns

### 17.1 Idempotency Key Pattern

```python
from typing import Any, Callable, Awaitable
from functools import wraps
import hashlib
import json


class IdempotencyStore:
    """Simple in-memory store. Use Redis in production."""
    def __init__(self):
        self._store: dict[str, Any] = {}

    def get(self, key: str) -> Any | None:
        return self._store.get(key)

    def set(self, key: str, value: Any) -> None:
        self._store[key] = value


def idempotent(store: IdempotencyStore):
    def decorator(func: Callable[..., Awaitable[Any]]):
        @wraps(func)
        async def wrapper(*args, idempotency_key: str, **kwargs):
            cached = store.get(idempotency_key)
            if cached is not None:
                return cached

            result = await func(*args, **kwargs)
            store.set(idempotency_key, result)
            return result
        return wrapper
    return decorator
```

### 17.2 Transactional Outbox (Simplified)

```python
from dataclasses import dataclass
from datetime import datetime
from typing import Any
import json


@dataclass(slots=True)
class OutboxMessage:
    id: str
    topic: str
    payload: dict[str, Any]
    created_at: datetime
    published: bool = False


class OutboxRepository:
    def __init__(self):
        self._messages: list[OutboxMessage] = []

    def add(self, message: OutboxMessage) -> None:
        self._messages.append(message)

    def get_unpublished(self) -> list[OutboxMessage]:
        return [m for m in self._messages if not m.published]

    def mark_published(self, message_id: str) -> None:
        for m in self._messages:
            if m.id == message_id:
                m.published = True
                break


# Usage pattern:
# 1. In the same DB transaction as business data, insert into outbox
# 2. Background worker reads unpublished messages and publishes to broker
# 3. Marks them as published
```

---

## 18. Advanced Typing Techniques

### 18.1 overload, TypeVarTuple, Unpack

```python
from typing import TypeVar, TypeVarTuple, Unpack, overload, Any

T = TypeVar("T")
Ts = TypeVarTuple("Ts")


@overload
def first(x: tuple[T, Unpack[Ts]]) -> T: ...
@overload
def first(x: list[T]) -> T: ...

def first(x: Any) -> Any:
    return x[0]
```

### 18.2 TypedDict with Required / NotRequired & total

```python
from typing import TypedDict, Required, NotRequired


class UserUpdate(TypedDict, total=False):
    email: str
    full_name: str
    is_active: bool


class CreateUser(TypedDict):
    email: Required[str]
    full_name: Required[str]
    is_active: NotRequired[bool]
```

### 18.3 ParamSpec + Concatenate (Decorator Signatures)

```python
from typing import Callable, ParamSpec, TypeVar, Concatenate
from functools import wraps

P = ParamSpec("P")
R = TypeVar("R")


def with_request_id(func: Callable[Concatenate[str, P], R]) -> Callable[P, R]:
    @wraps(func)
    def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
        request_id = "generated-id"
        return func(request_id, *args, **kwargs)
    return wrapper
```

---

## 19. Domain-Driven Design Tactics in Python

### 19.1 Value Objects

```python
from dataclasses import dataclass


@dataclass(frozen=True, slots=True)
class Money:
    amount: float
    currency: str = "USD"

    def __post_init__(self):
        if self.amount < 0:
            raise ValueError("Amount cannot be negative")
        object.__setattr__(self, "amount", round(self.amount, 2))

    def add(self, other: "Money") -> "Money":
        if self.currency != other.currency:
            raise ValueError("Currency mismatch")
        return Money(self.amount + other.amount, self.currency)
```

### 19.2 Aggregates & Domain Events (Lightweight)

```python
from dataclasses import dataclass, field
from datetime import datetime
from typing import List, Any
import uuid


@dataclass(slots=True)
class DomainEvent:
    event_id: str = field(default_factory=lambda: str(uuid.uuid4()))
    occurred_at: datetime = field(default_factory=datetime.utcnow)


@dataclass(slots=True)
class OrderCreated(DomainEvent):
    order_id: str
    customer_id: str
    total: float


class Order:
    def __init__(self, order_id: str, customer_id: str):
        self.order_id = order_id
        self.customer_id = customer_id
        self.items: list = []
        self._events: List[DomainEvent] = []

    def add_item(self, product_id: str, quantity: int, price: float) -> None:
        self.items.append({"product_id": product_id, "quantity": quantity, "price": price})

    def confirm(self) -> None:
        if not self.items:
            raise ValueError("Cannot confirm empty order")
        total = sum(i["quantity"] * i["price"] for i in self.items)
        self._events.append(
            OrderCreated(
                order_id=self.order_id,
                customer_id=self.customer_id,
                total=total,
            )
        )

    def collect_events(self) -> List[DomainEvent]:
        events = self._events[:]
        self._events.clear()
        return events
```

### 19.3 Specification Pattern

```python
from typing import Protocol, Any


class Specification(Protocol):
    def is_satisfied_by(self, candidate: Any) -> bool: ...


class ActiveUserSpec:
    def is_satisfied_by(self, user: Any) -> bool:
        return getattr(user, "is_active", False)


class EmailDomainSpec:
    def __init__(self, domain: str):
        self.domain = domain

    def is_satisfied_by(self, user: Any) -> bool:
        email = getattr(user, "email", "")
        return email.endswith(f"@{self.domain}")


class AndSpec:
    def __init__(self, *specs: Specification):
        self.specs = specs

    def is_satisfied_by(self, candidate: Any) -> bool:
        return all(s.is_satisfied_by(candidate) for s in self.specs)
```

---

## 20. Rate Limiting & Bulkheads

### 20.1 Token Bucket Rate Limiter (asyncio-friendly)

```python
import asyncio
import time


class AsyncRateLimiter:
    def __init__(self, rate: float, capacity: float):
        """
        rate: tokens per second
        capacity: maximum burst size
        """
        self.rate = rate
        self.capacity = capacity
        self.tokens = capacity
        self.updated_at = time.monotonic()
        self._lock = asyncio.Lock()

    async def acquire(self, tokens: float = 1.0) -> None:
        async with self._lock:
            now = time.monotonic()
            elapsed = now - self.updated_at
            self.tokens = min(self.capacity, self.tokens + elapsed * self.rate)
            self.updated_at = now

            if self.tokens < tokens:
                delay = (tokens - self.tokens) / self.rate
                await asyncio.sleep(delay)
                self.tokens = 0
            else:
                self.tokens -= tokens
```

### 20.2 Bulkhead (Isolate Resources)

```python
import asyncio
from typing import TypeVar, Callable, Awaitable

T = TypeVar("T")


class Bulkhead:
    """Limits concurrent executions for a specific dependency."""

    def __init__(self, max_concurrent: int):
        self._semaphore = asyncio.Semaphore(max_concurrent)

    async def run(self, coro: Awaitable[T]) -> T:
        async with self._semaphore:
            return await coro
```

---

## 21. Lazy Loading, Import Hooks & Plugin Discovery

### 21.1 Lazy Module Attributes

```python
# mypackage/__init__.py
from typing import Any

__all__ = ["heavy_function", "HeavyClass"]


def __getattr__(name: str) -> Any:
    if name == "heavy_function":
        from .heavy_module import heavy_function
        return heavy_function
    if name == "HeavyClass":
        from .heavy_module import HeavyClass
        return HeavyClass
    raise AttributeError(f"module {__name__!r} has no attribute {name!r}")
```

### 21.2 Entry-Point Based Plugins

```python
# pyproject.toml
# [project.entry-points."myapp.plugins"]
# csv = "myapp.plugins.csv:CSVPlugin"
# json = "myapp.plugins.json:JSONPlugin"

from importlib.metadata import entry_points


def load_plugins(group: str = "myapp.plugins") -> dict[str, type]:
    eps = entry_points().select(group=group)
    return {ep.name: ep.load() for ep in eps}
```

---

## 22. Buffer Protocol, Memoryview & Zero-Copy

```python
def process_large_binary(data: bytes) -> int:
    """
    Use memoryview to avoid copying large byte buffers.
    """
    view = memoryview(data)
    total = 0
    # Process in chunks without copying
    for i in range(0, len(view), 4096):
        chunk = view[i:i+4096]
        total += sum(chunk)
    return total


# Writing into a pre-allocated buffer
def fill_buffer(buf: bytearray, value: int) -> None:
    view = memoryview(buf)
    for i in range(len(view)):
        view[i] = value
```

Useful when:
- Working with large binary payloads
- Interfacing with C extensions / network protocols
- Implementing zero-copy parsing

---

## Summary Decision Table (Expanded)

| Concern                          | Recommended Approach                              |
|----------------------------------|---------------------------------------------------|
| Concurrent I/O                   | asyncio + TaskGroup + Semaphores                  |
| CPU-bound work                   | ProcessPoolExecutor / Numba / Cython              |
| Backpressure                     | asyncio.Queue(maxsize=...)                        |
| Memory-sensitive objects         | `__slots__` / dataclass(slots=True) / weakref     |
| Expected failures                | Result/Either or typed exceptions                 |
| Resilience                       | Circuit Breaker + Retry + Bulkhead + Rate Limiter |
| Distributed side-effects         | Idempotency keys + Transactional Outbox           |
| Configuration                    | Pydantic Settings                                 |
| Validation                       | Pydantic v2 (discriminated unions, validators)    |
| Caching                          | TTL + weakref + Redis                             |
| Observability                    | structlog + contextvars + OpenTelemetry           |
| Domain modeling                  | Value Objects + Aggregates + Domain Events        |
| Public library API               | Explicit `__all__` + deprecation + lazy imports   |
| Plugin systems                   | entry_points + `__init_subclass__`                |
| Zero-copy binary work            | memoryview + buffer protocol                     |
| Security                         | No pickle, validate everything, secret manager    |

---

**Final Advice**

Advanced tactics are force multipliers only when applied deliberately. Measure first, keep the majority of the codebase simple and boring, and introduce complexity only where it clearly reduces risk or improves performance under real load.

---

*Document version: 2.0 (Expanded)*  
*Examples target Python 3.11+ and modern tooling*
