# Comprehensive Guide to Object-Oriented Programming in Python
### Production-Ready Patterns & Full Examples

This guide covers modern, idiomatic, and production-grade Object-Oriented Programming in Python (3.11+). It emphasizes clarity, maintainability, type safety, and real-world patterns over academic purity.

---

## Table of Contents

1. [Core Class Design](#1-core-class-design)
2. [Inheritance vs Composition](#2-inheritance-vs-composition)
3. [Abstract Base Classes (ABCs)](#3-abstract-base-classes-abcs)
4. [Protocols (Structural Subtyping)](#4-protocols-structural-subtyping)
5. [Dataclasses & Modern Data Modeling](#5-dataclasses--modern-data-modeling)
6. [Properties, Descriptors & Encapsulation](#6-properties-descriptors--encapsulation)
7. [Magic Methods (Dunder Methods)](#7-magic-methods-dunder-methods)
8. [Class Methods, Static Methods & Alternative Constructors](#8-class-methods-static-methods--alternative-constructors)
9. [Mixins & Multiple Inheritance](#9-mixins--multiple-inheritance)
10. [`__init_subclass__` & Plugin Systems](#10-__init_subclass__--plugin-systems)
11. [Slots & Memory Optimization](#11-slots--memory-optimization)
12. [SOLID Principles in Python](#12-solid-principles-in-python)
13. [Common Design Patterns](#13-common-design-patterns)
14. [Production Architecture Patterns](#14-production-architecture-patterns)
15. [Best Practices & Anti-Patterns](#15-best-practices--anti-patterns)

---

## 1. Core Class Design

### 1.1 Clean, Typed Class Skeleton

```python
from __future__ import annotations
from typing import Optional
from datetime import datetime
import logging

logger = logging.getLogger(__name__)


class User:
    """
    Represents a user in the system.
    
    Prefer composition and clear public interfaces.
    Keep __init__ focused on essential state.
    """

    def __init__(
        self,
        user_id: str,
        email: str,
        full_name: str,
        is_active: bool = True,
        created_at: Optional[datetime] = None,
    ) -> None:
        self._user_id = user_id
        self._email = email
        self.full_name = full_name
        self.is_active = is_active
        self.created_at = created_at or datetime.utcnow()

    @property
    def user_id(self) -> str:
        return self._user_id

    @property
    def email(self) -> str:
        return self._email

    def deactivate(self) -> None:
        """Business method — prefer expressive verbs over setters."""
        if not self.is_active:
            logger.warning("User %s is already inactive", self.user_id)
            return
        self.is_active = False
        logger.info("User %s deactivated", self.user_id)

    def __repr__(self) -> str:
        return (
            f"User(user_id={self.user_id!r}, email={self.email!r}, "
            f"is_active={self.is_active})"
        )

    def __eq__(self, other: object) -> bool:
        if not isinstance(other, User):
            return NotImplemented
        return self.user_id == other.user_id
```

**Production notes:**
- Use private attributes (`_name`) for internal state.
- Prefer methods that express business intent (`deactivate()`) over generic setters.
- Always implement a useful `__repr__`.
- Make equality explicit and meaningful.

---

## 2. Inheritance vs Composition

### 2.1 Prefer Composition

```python
from dataclasses import dataclass, field
from typing import Protocol


class Notifier(Protocol):
    def send(self, message: str) -> None: ...


class EmailNotifier:
    def send(self, message: str) -> None:
        print(f"Sending email: {message}")


class SMSNotifier:
    def send(self, message: str) -> None:
        print(f"Sending SMS: {message}")


@dataclass
class OrderService:
    notifier: Notifier
    # Composition: OrderService *has a* Notifier

    def place_order(self, order_id: str) -> None:
        # business logic...
        self.notifier.send(f"Order {order_id} placed successfully")


# Usage — easy to swap implementations
email_service = OrderService(notifier=EmailNotifier())
sms_service = OrderService(notifier=SMSNotifier())
```

### 2.2 When Inheritance Is Acceptable

Use inheritance only for true **"is-a"** relationships and shared behavior that benefits from polymorphism.

```python
from abc import ABC, abstractmethod
from typing import List


class Shape(ABC):
    @abstractmethod
    def area(self) -> float:
        ...

    @abstractmethod
    def perimeter(self) -> float:
        ...


class Rectangle(Shape):
    def __init__(self, width: float, height: float) -> None:
        self.width = width
        self.height = height

    def area(self) -> float:
        return self.width * self.height

    def perimeter(self) -> float:
        return 2 * (self.width + self.height)


class Circle(Shape):
    def __init__(self, radius: float) -> None:
        self.radius = radius

    def area(self) -> float:
        return 3.14159 * self.radius ** 2

    def perimeter(self) -> float:
        return 2 * 3.14159 * self.radius
```

---

## 3. Abstract Base Classes (ABCs)

ABCs enforce a nominal interface and can provide shared implementation.

```python
from abc import ABC, abstractmethod
from typing import List, Optional
from datetime import datetime


class Repository(ABC):
    """Generic repository interface with some shared behavior."""

    @abstractmethod
    def get(self, entity_id: str) -> Optional[object]:
        ...

    @abstractmethod
    def save(self, entity: object) -> None:
        ...

    @abstractmethod
    def delete(self, entity_id: str) -> None:
        ...

    def get_or_raise(self, entity_id: str) -> object:
        """Concrete method provided by the ABC."""
        entity = self.get(entity_id)
        if entity is None:
            raise ValueError(f"Entity {entity_id} not found")
        return entity


class InMemoryUserRepository(Repository):
    def __init__(self) -> None:
        self._storage: dict[str, User] = {}

    def get(self, entity_id: str) -> Optional[User]:
        return self._storage.get(entity_id)

    def save(self, entity: User) -> None:
        self._storage[entity.user_id] = entity

    def delete(self, entity_id: str) -> None:
        self._storage.pop(entity_id, None)
```

**When to use ABCs vs Protocols:**
- Use **ABC** when you need to enforce inheritance or provide shared implementation.
- Use **Protocol** when you only care about the shape of the object (duck typing + static checking).

---

## 4. Protocols (Structural Subtyping)

Preferred modern approach for interfaces in many cases.

```python
from typing import Protocol, runtime_checkable, Iterable, Optional
from typing import TypeVar

T = TypeVar("T")


@runtime_checkable
class Readable(Protocol):
    def read(self, size: int = -1) -> bytes: ...


@runtime_checkable
class Repository(Protocol[T]):
    def get(self, id: str) -> Optional[T]: ...
    def save(self, entity: T) -> None: ...
    def list(self) -> Iterable[T]: ...


def process_entities(repo: Repository[User]) -> list[User]:
    return [u for u in repo.list() if u.is_active]


# Any class that implements the methods satisfies the Protocol
# — no explicit inheritance required.
```

---

## 5. Dataclasses & Modern Data Modeling

```python
from dataclasses import dataclass, field, asdict
from typing import List, ClassVar, Optional
from datetime import datetime
from enum import Enum, auto


class OrderStatus(Enum):
    PENDING = auto()
    CONFIRMED = auto()
    SHIPPED = auto()
    DELIVERED = auto()
    CANCELLED = auto()


@dataclass(slots=True, kw_only=True)
class OrderItem:
    product_id: str
    quantity: int
    unit_price: float

    @property
    def total(self) -> float:
        return self.quantity * self.unit_price


@dataclass(slots=True, kw_only=True)
class Order:
    order_id: str
    customer_id: str
    items: List[OrderItem] = field(default_factory=list)
    status: OrderStatus = OrderStatus.PENDING
    created_at: datetime = field(default_factory=datetime.utcnow)
    notes: Optional[str] = None

    # Class-level constant
    MAX_ITEMS: ClassVar[int] = 50

    def __post_init__(self) -> None:
        if len(self.items) > self.MAX_ITEMS:
            raise ValueError(f"Order cannot exceed {self.MAX_ITEMS} items")
        if not self.items:
            raise ValueError("Order must contain at least one item")

    @property
    def total_amount(self) -> float:
        return sum(item.total for item in self.items)

    def confirm(self) -> None:
        if self.status != OrderStatus.PENDING:
            raise RuntimeError(f"Cannot confirm order in status {self.status}")
        self.status = OrderStatus.CONFIRMED

    def to_dict(self) -> dict:
        return asdict(self)
```

**Key advantages of `slots=True`:**
- Lower memory usage
- Faster attribute access
- Prevents accidental creation of new attributes

---

## 6. Properties, Descriptors & Encapsulation

### 6.1 Properties for Computed Values & Validation

```python
class BankAccount:
    def __init__(self, owner: str, balance: float = 0.0) -> None:
        self.owner = owner
        self._balance = balance

    @property
    def balance(self) -> float:
        return self._balance

    @balance.setter
    def balance(self, value: float) -> None:
        if value < 0:
            raise ValueError("Balance cannot be negative")
        self._balance = value

    def deposit(self, amount: float) -> None:
        if amount <= 0:
            raise ValueError("Deposit amount must be positive")
        self.balance += amount

    def withdraw(self, amount: float) -> None:
        if amount <= 0:
            raise ValueError("Withdrawal amount must be positive")
        if amount > self.balance:
            raise ValueError("Insufficient funds")
        self.balance -= amount
```

### 6.2 Reusable Descriptor for Validation

```python
from typing import Any, Callable, Type, Optional


class ValidatedAttribute:
    def __init__(self, validator: Callable[[Any], None]) -> None:
        self.validator = validator
        self.name: str = ""

    def __set_name__(self, owner: Type, name: str) -> None:
        self.name = name
        self.private_name = f"_{name}"

    def __get__(self, obj: Any, objtype: Optional[Type] = None) -> Any:
        if obj is None:
            return self
        return getattr(obj, self.private_name)

    def __set__(self, obj: Any, value: Any) -> None:
        self.validator(value)
        setattr(obj, self.private_name, value)


def positive_number(value: float) -> None:
    if value <= 0:
        raise ValueError("Value must be positive")


class Product:
    price = ValidatedAttribute(positive_number)
    stock = ValidatedAttribute(lambda v: None if v >= 0 else (_ for _ in ()).throw(ValueError("Stock cannot be negative")))

    def __init__(self, name: str, price: float, stock: int) -> None:
        self.name = name
        self.price = price
        self.stock = stock
```

---

## 7. Magic Methods (Dunder Methods)

Implement only the ones you need. Most important production ones:

```python
from typing import Any


class Money:
    def __init__(self, amount: float, currency: str = "USD") -> None:
        self.amount = round(amount, 2)
        self.currency = currency

    def __repr__(self) -> str:
        return f"Money({self.amount}, {self.currency!r})"

    def __str__(self) -> str:
        return f"{self.amount:.2f} {self.currency}"

    def __eq__(self, other: object) -> bool:
        if not isinstance(other, Money):
            return NotImplemented
        return self.amount == other.amount and self.currency == other.currency

    def __lt__(self, other: object) -> bool:
        if not isinstance(other, Money):
            return NotImplemented
        if self.currency != other.currency:
            raise ValueError("Cannot compare different currencies")
        return self.amount < other.amount

    def __add__(self, other: object) -> Money:
        if not isinstance(other, Money):
            return NotImplemented
        if self.currency != other.currency:
            raise ValueError("Cannot add different currencies")
        return Money(self.amount + other.amount, self.currency)

    def __hash__(self) -> int:
        return hash((self.amount, self.currency))

    # Support for total_ordering if needed
    def __bool__(self) -> bool:
        return self.amount != 0
```

**Recommended dunders for domain objects:**
- `__repr__`, `__str__`
- `__eq__` + `__hash__` (if immutable)
- Comparison methods (`__lt__`, etc.) when ordering makes sense
- `__add__`, `__mul__`, etc. only for value objects

---

## 8. Class Methods, Static Methods & Alternative Constructors

```python
from datetime import datetime
from typing import Optional
import json


class User:
    def __init__(self, user_id: str, email: str, created_at: datetime) -> None:
        self.user_id = user_id
        self.email = email
        self.created_at = created_at

    @classmethod
    def from_dict(cls, data: dict) -> User:
        """Alternative constructor from dictionary."""
        return cls(
            user_id=data["user_id"],
            email=data["email"],
            created_at=datetime.fromisoformat(data["created_at"]),
        )

    @classmethod
    def from_json(cls, json_str: str) -> User:
        return cls.from_dict(json.loads(json_str))

    @staticmethod
    def validate_email(email: str) -> bool:
        """Utility that doesn't need class or instance state."""
        return "@" in email and "." in email.split("@")[-1]

    def to_dict(self) -> dict:
        return {
            "user_id": self.user_id,
            "email": self.email,
            "created_at": self.created_at.isoformat(),
        }
```

---

## 9. Mixins & Multiple Inheritance

Use mixins for reusable behavior. Keep them focused and stateless when possible.

```python
from typing import Any
import json
import logging

logger = logging.getLogger(__name__)


class JSONSerializableMixin:
    """Mixin that adds JSON serialization capability."""

    def to_json(self, **kwargs: Any) -> str:
        return json.dumps(self.to_dict(), **kwargs)

    def to_dict(self) -> dict:
        raise NotImplementedError("Subclasses must implement to_dict()")


class LoggingMixin:
    """Mixin that adds structured logging helpers."""

    def log_info(self, message: str, **extra: Any) -> None:
        logger.info(message, extra={"class": self.__class__.__name__, **extra})

    def log_error(self, message: str, **extra: Any) -> None:
        logger.error(message, extra={"class": self.__class__.__name__, **extra})


class AuditMixin:
    created_at: datetime
    updated_at: datetime

    def touch(self) -> None:
        self.updated_at = datetime.utcnow()


@dataclass
class Document(JSONSerializableMixin, LoggingMixin, AuditMixin):
    doc_id: str
    title: str
    content: str
    created_at: datetime = field(default_factory=datetime.utcnow)
    updated_at: datetime = field(default_factory=datetime.utcnow)

    def to_dict(self) -> dict:
        return {
            "doc_id": self.doc_id,
            "title": self.title,
            "content": self.content,
            "created_at": self.created_at.isoformat(),
            "updated_at": self.updated_at.isoformat(),
        }
```

**MRO reminder:** Python uses C3 linearization. Use `ClassName.__mro__` to inspect.

---

## 10. `__init_subclass__` & Plugin Systems

Modern replacement for many metaclass use cases.

```python
from typing import ClassVar, Dict, Type, Any


class PluginBase:
    _registry: ClassVar[Dict[str, Type[PluginBase]]] = {}

    def __init_subclass__(cls, name: str | None = None, **kwargs: Any) -> None:
        super().__init_subclass__(**kwargs)
        plugin_name = name or cls.__name__
        
        if plugin_name in PluginBase._registry:
            raise RuntimeError(f"Plugin '{plugin_name}' already registered")
        
        PluginBase._registry[plugin_name] = cls
        cls.plugin_name = plugin_name  # type: ignore

    @classmethod
    def available_plugins(cls) -> list[str]:
        return list(cls._registry.keys())

    @classmethod
    def create(cls, name: str, **kwargs: Any) -> PluginBase:
        try:
            plugin_cls = cls._registry[name]
        except KeyError:
            raise ValueError(
                f"Unknown plugin '{name}'. Available: {cls.available_plugins()}"
            )
        return plugin_cls(**kwargs)


class CSVExporter(PluginBase, name="csv"):
    def __init__(self, delimiter: str = ",") -> None:
        self.delimiter = delimiter

    def export(self, data: list[dict]) -> str:
        # implementation...
        return "csv data"


class JSONExporter(PluginBase, name="json"):
    def export(self, data: list[dict]) -> str:
        return json.dumps(data)
```

---

## 11. Slots & Memory Optimization

```python
from dataclasses import dataclass
import sys


@dataclass(slots=True)
class Point:
    x: float
    y: float
    z: float = 0.0


@dataclass  # without slots
class PointRegular:
    x: float
    y: float
    z: float = 0.0


# Rough comparison
p_slots = Point(1.0, 2.0, 3.0)
p_regular = PointRegular(1.0, 2.0, 3.0)

print(sys.getsizeof(p_slots))    # Significantly smaller
print(sys.getsizeof(p_regular))
```

Use `slots=True` when:
- You create many instances
- Memory is a concern
- You want to prevent dynamic attribute creation

---

## 12. SOLID Principles in Python

### Single Responsibility
```python
# Bad: God class
class UserManager:
    def create_user(...): ...
    def send_email(...): ...
    def generate_report(...): ...
    def log_activity(...): ...

# Good: Separated concerns
class UserService:
    def create_user(...): ...

class EmailService:
    def send(...): ...

class ReportGenerator:
    def generate(...): ...
```

### Open/Closed
Prefer extension via composition or protocols over modifying existing classes.

### Liskov Substitution
Subtypes must be substitutable for their base types without breaking behavior.

### Interface Segregation
Prefer small, focused protocols/ABCs over large ones.

### Dependency Inversion
Depend on abstractions (Protocols/ABCs), not concrete classes.

```python
class OrderProcessor:
    def __init__(self, notifier: Notifier, repo: Repository[Order]) -> None:
        self.notifier = notifier
        self.repo = repo
    # High-level module depends on abstractions
```

---

## 13. Common Design Patterns

### Strategy Pattern

```python
from typing import Protocol


class PaymentStrategy(Protocol):
    def pay(self, amount: float) -> bool: ...


class CreditCardPayment:
    def pay(self, amount: float) -> bool:
        print(f"Paying {amount} via Credit Card")
        return True


class PayPalPayment:
    def pay(self, amount: float) -> bool:
        print(f"Paying {amount} via PayPal")
        return True


class PaymentService:
    def __init__(self, strategy: PaymentStrategy) -> None:
        self.strategy = strategy

    def process_payment(self, amount: float) -> bool:
        return self.strategy.pay(amount)
```

### Factory Pattern

```python
from typing import Type


class NotifierFactory:
    _strategies: dict[str, Type[Notifier]] = {
        "email": EmailNotifier,
        "sms": SMSNotifier,
    }

    @classmethod
    def create(cls, kind: str) -> Notifier:
        try:
            return cls._strategies[kind]()
        except KeyError:
            raise ValueError(f"Unknown notifier type: {kind}")
```

### Observer (Simple)

```python
from typing import List, Callable, Any


class EventEmitter:
    def __init__(self) -> None:
        self._listeners: dict[str, List[Callable[..., Any]]] = {}

    def on(self, event: str, callback: Callable[..., Any]) -> None:
        self._listeners.setdefault(event, []).append(callback)

    def emit(self, event: str, *args: Any, **kwargs: Any) -> None:
        for callback in self._listeners.get(event, []):
            callback(*args, **kwargs)
```

---

## 14. Production Architecture Patterns

### Repository + Service Layer

```python
class UserRepository(Protocol):
    def get(self, user_id: str) -> User | None: ...
    def save(self, user: User) -> None: ...


class UserService:
    def __init__(self, repo: UserRepository, notifier: Notifier) -> None:
        self.repo = repo
        self.notifier = notifier

    def register_user(self, email: str, full_name: str) -> User:
        if self.repo.get_by_email(email):
            raise ValueError("Email already registered")
        
        user = User.create(email=email, full_name=full_name)
        self.repo.save(user)
        self.notifier.send(f"Welcome {full_name}!")
        return user

    def deactivate_user(self, user_id: str) -> None:
        user = self.repo.get(user_id)
        if user is None:
            raise ValueError("User not found")
        user.deactivate()
        self.repo.save(user)
```

### Dependency Injection Friendly Design

All dependencies are received via constructor. This makes testing and composition trivial.

---

## 15. Best Practices & Anti-Patterns

### Do

- Prefer composition over inheritance
- Use Protocols for interfaces in most cases
- Make illegal states unrepresentable (enums, validation in `__post_init__`)
- Keep classes focused (Single Responsibility)
- Implement `__repr__` on almost every class
- Use `slots=True` for high-volume data classes
- Favor explicit over implicit
- Depend on abstractions

### Avoid

- Deep inheritance hierarchies
- God classes
- Anemic domain models (data + no behavior) *or* the opposite extreme
- Overusing metaclasses
- Mutable default arguments
- Catching generic `Exception` inside domain logic
- Making everything a singleton
- Leaking infrastructure concerns into domain objects

---

## Quick Reference: Choosing the Right Tool

| Need                              | Recommended Approach                  |
|-----------------------------------|---------------------------------------|
| Data container                    | `@dataclass(slots=True)`              |
| Interface / contract              | `Protocol` (preferred) or ABC         |
| Shared behavior across classes    | Mixin or composition                  |
| Plugin / registry system          | `__init_subclass__`                   |
| Attribute validation              | Descriptor or Pydantic                |
| Alternative constructors          | `@classmethod`                        |
| Utility with no state             | `@staticmethod` or module function    |
| High instance count               | `slots=True`                          |
| Cross-cutting concerns            | Decorator or Mixin                    |

---

**Final Recommendation**

Write simple, explicit classes first. Introduce inheritance, descriptors, or metaclasses only when the complexity is clearly justified by the problem. The best Python OOP code is often the most boring and readable.

---

*Target Python version: 3.11+*  
*Style: Production-oriented, type-hinted, modern idioms*
