# Advanced Python Development Techniques
### Production Patterns & Full Examples

This guide expands on advanced Python techniques with complete, production-oriented examples, best practices, and common pitfalls.

---

## 1. Decorators — Production Patterns

### 1.1 Robust Retry Decorator (Exponential Backoff + Jitter)

```python
from functools import wraps
import time
import random
import logging
from typing import Callable, Type, Tuple, Optional

logger = logging.getLogger(__name__)

def retry(
    max_attempts: int = 3,
    base_delay: float = 1.0,
    max_delay: float = 60.0,
    exponential_base: float = 2.0,
    jitter: bool = True,
    exceptions: Tuple[Type[Exception], ...] = (Exception,),
    on_retry: Optional[Callable] = None,
):
    """
    Production-grade retry decorator with exponential backoff and jitter.
    
    Args:
        max_attempts: Maximum number of attempts
        base_delay: Initial delay in seconds
        max_delay: Maximum delay between retries
        exponential_base: Multiplier for exponential backoff
        jitter: Add randomness to avoid thundering herd
        exceptions: Exception types that trigger a retry
        on_retry: Optional callback(attempt, exception, delay)
    """
    def decorator(func: Callable):
        @wraps(func)
        def wrapper(*args, **kwargs):
            last_exception = None
            
            for attempt in range(1, max_attempts + 1):
                try:
                    return func(*args, **kwargs)
                except exceptions as e:
                    last_exception = e
                    
                    if attempt == max_attempts:
                        logger.error(
                            f"{func.__name__} failed after {max_attempts} attempts: {e}"
                        )
                        raise
                    
                    # Calculate delay with exponential backoff
                    delay = min(base_delay * (exponential_base ** (attempt - 1)), max_delay)
                    
                    if jitter:
                        delay = delay * (0.5 + random.random())  # 50-150% of calculated delay
                    
                    logger.warning(
                        f"{func.__name__} failed (attempt {attempt}/{max_attempts}): {e}. "
                        f"Retrying in {delay:.2f}s..."
                    )
                    
                    if on_retry:
                        on_retry(attempt, e, delay)
                    
                    time.sleep(delay)
            
            raise last_exception  # unreachable, but keeps type checkers happy
        
        return wrapper
    return decorator


# Usage example
@retry(max_attempts=5, base_delay=0.5, exceptions=(ConnectionError, TimeoutError))
def fetch_remote_data(url: str) -> dict:
    # Simulate flaky network call
    ...
```

### 1.2 Timing + Metrics Decorator

```python
from functools import wraps
import time
from typing import Callable
import logging

logger = logging.getLogger(__name__)

def timed(metric_name: str | None = None):
    def decorator(func: Callable):
        name = metric_name or f"{func.__module__}.{func.__qualname__}"
        
        @wraps(func)
        def wrapper(*args, **kwargs):
            start = time.perf_counter()
            try:
                result = func(*args, **kwargs)
                status = "success"
                return result
            except Exception:
                status = "error"
                raise
            finally:
                elapsed = time.perf_counter() - start
                logger.info(
                    "function_timing",
                    extra={
                        "metric": name,
                        "duration_ms": round(elapsed * 1000, 2),
                        "status": status,
                    }
                )
                # In real systems: push to Prometheus, StatsD, OpenTelemetry, etc.
        
        return wrapper
    return decorator
```

### 1.3 Class Decorator for Automatic Registration

```python
from typing import Type, Dict

PLUGIN_REGISTRY: Dict[str, Type] = {}

def register_plugin(name: str | None = None):
    def decorator(cls: Type):
        plugin_name = name or cls.__name__
        if plugin_name in PLUGIN_REGISTRY:
            raise ValueError(f"Plugin '{plugin_name}' already registered")
        PLUGIN_REGISTRY[plugin_name] = cls
        cls.plugin_name = plugin_name  # attach metadata
        return cls
    return decorator


@register_plugin("csv_exporter")
class CSVExporter:
    def export(self, data): ...
```

---

## 2. Context Managers — Production Patterns

### 2.1 Database Transaction Context Manager

```python
from contextlib import contextmanager
from typing import Iterator
import logging

logger = logging.getLogger(__name__)

@contextmanager
def transaction(connection) -> Iterator:
    """
    Ensures commit/rollback semantics.
    Works with DB-API 2.0 connections.
    """
    original_autocommit = getattr(connection, "autocommit", None)
    
    try:
        if original_autocommit is not None:
            connection.autocommit = False
        
        yield connection
        
        connection.commit()
        logger.debug("Transaction committed")
    except Exception:
        connection.rollback()
        logger.exception("Transaction rolled back due to exception")
        raise
    finally:
        if original_autocommit is not None:
            connection.autocommit = original_autocommit
```

### 2.2 Resource Acquisition with Timeout + Cleanup

```python
from contextlib import contextmanager
import time
from typing import Iterator, Any

class ResourcePool:
    def acquire(self, timeout: float = 5.0) -> Any: ...
    def release(self, resource: Any) -> None: ...


@contextmanager
def pooled_resource(pool: ResourcePool, timeout: float = 5.0) -> Iterator[Any]:
    resource = None
    try:
        resource = pool.acquire(timeout=timeout)
        yield resource
    finally:
        if resource is not None:
            pool.release(resource)
```

### 2.3 Multiple Resources with ExitStack

```python
from contextlib import ExitStack, contextmanager

@contextmanager
def multi_resource_manager(resources):
    with ExitStack() as stack:
        acquired = [stack.enter_context(r) for r in resources]
        yield acquired
```

---

## 3. Generators & Streaming Pipelines

### 3.1 Memory-Efficient ETL Pipeline

```python
from typing import Iterator, Iterable, Callable, Any
import logging

logger = logging.getLogger(__name__)

def read_records(source: str) -> Iterator[dict]:
    """Yield records lazily from a large file/stream."""
    with open(source, "r") as f:
        for line_num, line in enumerate(f, 1):
            try:
                yield parse_line(line)
            except Exception as e:
                logger.warning(f"Skipping malformed line {line_num}: {e}")


def transform(records: Iterable[dict], *transformers: Callable[[dict], dict]) -> Iterator[dict]:
    for record in records:
        for transformer in transformers:
            record = transformer(record)
        yield record


def filter_records(records: Iterable[dict], predicate: Callable[[dict], bool]) -> Iterator[dict]:
    for record in records:
        if predicate(record):
            yield record


def batch(records: Iterable[dict], size: int = 1000) -> Iterator[list[dict]]:
    """Yield lists of records for bulk operations."""
    batch_list = []
    for record in records:
        batch_list.append(record)
        if len(batch_list) >= size:
            yield batch_list
            batch_list = []
    if batch_list:
        yield batch_list


# Usage
pipeline = batch(
    filter_records(
        transform(
            read_records("large_dataset.csv"),
            normalize_keys,
            convert_types,
            enrich_with_geodata,
        ),
        lambda r: r["status"] == "active",
    ),
    size=500,
)

for chunk in pipeline:
    bulk_insert(chunk)
```

### 3.2 `yield from` for Recursive Flattening + Async Generator

```python
from typing import Iterable, Any, AsyncIterator
import asyncio

def flatten(items: Iterable[Any]) -> Iterator[Any]:
    for item in items:
        if isinstance(item, (list, tuple, set)):
            yield from flatten(item)
        else:
            yield item


async def async_chunked_reader(stream, chunk_size: int = 1024) -> AsyncIterator[bytes]:
    while True:
        chunk = await stream.read(chunk_size)
        if not chunk:
            break
        yield chunk
```

---

## 4. Descriptors — Real-World Validation & ORM-style Fields

```python
from typing import Any, Callable, Type, Optional
import re

class Validator:
    def __init__(self, *validators: Callable[[Any], None]):
        self.validators = validators

    def __set_name__(self, owner: Type, name: str):
        self.name = name
        self.private_name = f"_{name}"

    def __get__(self, obj: Any, objtype: Optional[Type] = None) -> Any:
        if obj is None:
            return self
        return getattr(obj, self.private_name, None)

    def __set__(self, obj: Any, value: Any) -> None:
        for validator in self.validators:
            validator(value)
        setattr(obj, self.private_name, value)


def typed(expected_type: Type):
    def validator(value):
        if not isinstance(value, expected_type):
            raise TypeError(f"Expected {expected_type.__name__}, got {type(value).__name__}")
    return validator


def ranged(min_value: float | None = None, max_value: float | None = None):
    def validator(value):
        if min_value is not None and value < min_value:
            raise ValueError(f"Value {value} is below minimum {min_value}")
        if max_value is not None and value > max_value:
            raise ValueError(f"Value {value} is above maximum {max_value}")
    return validator


def matched(pattern: str):
    regex = re.compile(pattern)
    def validator(value: str):
        if not regex.fullmatch(value):
            raise ValueError(f"Value '{value}' does not match pattern {pattern}")
    return validator


class User:
    name = Validator(typed(str), matched(r"^[A-Za-z ]{2,50}$"))
    age = Validator(typed(int), ranged(0, 150))
    email = Validator(typed(str), matched(r"^[^@]+@[^@]+\.[^@]+$"))

    def __init__(self, name: str, age: int, email: str):
        self.name = name
        self.age = age
        self.email = email
```

---

## 5. `__init_subclass__` vs Metaclasses (Preferred Modern Approach)

```python
from typing import Dict, Type, ClassVar

class PluginBase:
    _registry: ClassVar[Dict[str, Type["PluginBase"]]] = {}

    def __init_subclass__(cls, plugin_name: str | None = None, **kwargs):
        super().__init_subclass__(**kwargs)
        
        name = plugin_name or cls.__name__
        if name in PluginBase._registry:
            raise RuntimeError(f"Duplicate plugin name: {name}")
        
        PluginBase._registry[name] = cls
        cls.plugin_name = name

    @classmethod
    def get_plugin(cls, name: str) -> Type["PluginBase"]:
        try:
            return cls._registry[name]
        except KeyError:
            raise ValueError(f"Unknown plugin: {name}. Available: {list(cls._registry)}")


class CSVExporter(PluginBase, plugin_name="csv"):
    def export(self, data): ...


class JSONExporter(PluginBase, plugin_name="json"):
    def export(self, data): ...
```

---

## 6. Advanced Typing Patterns

### 6.1 Protocols + Structural Subtyping

```python
from typing import Protocol, runtime_checkable, Iterator, TypeVar
from collections.abc import Iterable

T = TypeVar("T")

@runtime_checkable
class SupportsClose(Protocol):
    def close(self) -> None: ...


@runtime_checkable
class Repository(Protocol[T]):
    def get(self, id: str) -> T | None: ...
    def save(self, entity: T) -> None: ...
    def delete(self, id: str) -> None: ...
    def list(self) -> Iterable[T]: ...


def process_all(repo: Repository[T]) -> list[T]:
    return list(repo.list())
```

### 6.2 ParamSpec for Decorators (Preserving Signatures)

```python
from typing import Callable, ParamSpec, TypeVar, Any
from functools import wraps

P = ParamSpec("P")
R = TypeVar("R")

def log_calls(func: Callable[P, R]) -> Callable[P, R]:
    @wraps(func)
    def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
        print(f"Calling {func.__name__}")
        return func(*args, **kwargs)
    return wrapper
```

---

## 7. High-Performance Data Classes & Memory Optimization

```python
from dataclasses import dataclass, field
from typing import ClassVar
import sys

@dataclass(slots=True, frozen=True, weakref_slot=True)
class Point:
    x: float
    y: float
    z: float = 0.0

    # Class-level constant
    ORIGIN: ClassVar["Point"]

    def distance_to(self, other: "Point") -> float:
        return ((self.x - other.x)**2 + (self.y - other.y)**2 + (self.z - other.z)**2) ** 0.5


Point.ORIGIN = Point(0.0, 0.0, 0.0)

# Memory comparison demonstration
print(sys.getsizeof(Point(1, 2, 3)))  # Significantly smaller than regular class instances
```

---

## 8. Structured Concurrency with asyncio (Python 3.11+)

```python
import asyncio
import logging
from typing import Any
from contextlib import asynccontextmanager

logger = logging.getLogger(__name__)

async def fetch_user(user_id: int) -> dict:
    await asyncio.sleep(0.1)  # simulate I/O
    return {"id": user_id, "name": f"User {user_id}"}


async def fetch_orders(user_id: int) -> list[dict]:
    await asyncio.sleep(0.2)
    return [{"order_id": 1, "user_id": user_id}]


async def load_user_dashboard(user_id: int) -> dict:
    """
    Structured concurrency example using TaskGroup.
    If any task fails, all others are cancelled.
    """
    async with asyncio.TaskGroup() as tg:
        user_task = tg.create_task(fetch_user(user_id))
        orders_task = tg.create_task(fetch_orders(user_id))
    
    # All tasks completed successfully
    return {
        "user": user_task.result(),
        "orders": orders_task.result(),
    }


@asynccontextmanager
async def managed_session(session_factory):
    session = session_factory()
    try:
        yield session
        await session.commit()
    except Exception:
        await session.rollback()
        raise
    finally:
        await session.close()
```

---

## 9. Production Packaging Layout (`src/` + `pyproject.toml`)

```toml
# pyproject.toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "my-advanced-lib"
version = "0.1.0"
description = "Production-grade Python library"
readme = "README.md"
requires-python = ">=3.11"
license = "MIT"
authors = [
    { name = "Your Name", email = "you@example.com" }
]
dependencies = [
    "httpx>=0.27",
    "pydantic>=2.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0",
    "pytest-asyncio>=0.23",
    "mypy>=1.8",
    "ruff>=0.3",
    "hypothesis>=6.0",
]

[project.scripts]
mycli = "my_advanced_lib.cli:main"

[tool.hatch.build.targets.wheel]
packages = ["src/my_advanced_lib"]

[tool.ruff]
line-length = 100
target-version = "py311"

[tool.mypy]
python_version = "3.11"
strict = true
warn_return_any = true
warn_unused_configs = true

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
```

Recommended layout:

```
my-project/
├── pyproject.toml
├── README.md
├── src/
│   └── my_advanced_lib/
│       ├── __init__.py
│       ├── core.py
│       ├── plugins/
│       └── py.typed          # marks package as typed
└── tests/
    ├── conftest.py
    └── test_core.py
```

---

## 10. Advanced Testing Patterns (pytest)

```python
import pytest
from hypothesis import given, strategies as st
from unittest.mock import AsyncMock, patch

# Parametrized + fixture composition
@pytest.fixture
def db_session():
    session = create_test_session()
    yield session
    session.rollback()
    session.close()


@pytest.mark.parametrize("input_value,expected", [
    (0, 0),
    (1, 1),
    (10, 55),
    (-1, None),
])
def test_fibonacci(input_value, expected):
    if expected is None:
        with pytest.raises(ValueError):
            fibonacci(input_value)
    else:
        assert fibonacci(input_value) == expected


# Property-based testing
@given(st.lists(st.integers(), min_size=0, max_size=1000))
def test_sort_idempotent(data):
    assert sorted(sorted(data)) == sorted(data)


# Async testing
@pytest.mark.asyncio
async def test_load_dashboard():
    with patch("myapp.services.fetch_user", new_callable=AsyncMock) as mock_user:
        mock_user.return_value = {"id": 1, "name": "Test"}
        result = await load_user_dashboard(1)
        assert result["user"]["name"] == "Test"
```

---

## Summary of Production Recommendations

| Technique              | When to Use                              | Preferred Modern Alternative      | Risk Level |
|------------------------|------------------------------------------|-----------------------------------|------------|
| Decorators             | Cross-cutting concerns                   | —                                 | Low        |
| Context Managers       | Resource lifecycle                       | —                                 | Low        |
| Generators             | Large data / streaming                   | —                                 | Low        |
| Descriptors            | Reusable attribute logic                 | `__init_subclass__` + properties  | Medium     |
| Metaclasses            | Deep class creation control              | `__init_subclass__`               | High       |
| Protocols              | Duck typing with static checks           | ABCs (when nominal needed)        | Low        |
| `slots=True` dataclass | High-volume objects                      | Regular classes                   | Low        |
| TaskGroup              | Concurrent async work                    | `asyncio.gather`                  | Low        |
| ParamSpec              | Signature-preserving decorators          | `Any` (loses type safety)         | Low        |

---

**Key Principles for Production Code**

1. Prefer `__init_subclass__` over metaclasses.
2. Always use `@functools.wraps` and `ParamSpec` for decorators.
3. Prefer structured concurrency (`TaskGroup`) over manual task management.
4. Make dynamic behavior explicit and well-documented.
5. Combine runtime validation (descriptors/Pydantic) with static typing.
6. Keep magic localized; the rest of the codebase should remain boring and readable.

---

*Document generated for advanced Python practitioners. Examples target Python 3.11+.*
