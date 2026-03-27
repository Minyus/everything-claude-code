---
name: python-patterns
description: Pythonic idioms, modern type hints, and best practices for building robust, efficient, and maintainable Python 3.11+ applications.
origin: ECC
---

# Python Development Patterns

Idiomatic Python patterns and best practices for Python 3.11+.

## When to Activate

- Writing new Python code
- Reviewing Python code
- Refactoring existing Python code
- Designing Python packages/modules

## Core Principles

### 1. Readability Counts

Python prioritizes readability. Code should be obvious and easy to understand.

### 2. Explicit is Better Than Implicit

Avoid magic; be clear about what your code does.

### 3. EAFP - Easier to Ask Forgiveness Than Permission

Python prefers exception handling over checking conditions.

## Type Hints (Python 3.11+)

### Use Built-in Types and `|` Unions

Never import `Optional`, `List`, `Dict`, `Union`, `Tuple` from `typing` - use built-ins and `|`.

```python
# Good: Modern type hints
def process_user(
    user_id: str,
    data: dict[str, Any],
    active: bool = True,
) -> User | None:
    """Process a user and return the updated User or None."""
    if not active:
        return None
    return User(user_id, data)


def process_items(items: list[str]) -> dict[str, int]:
    return {item: len(item) for item in items}


# Bad: Legacy typing imports (Python < 3.9 style)
from typing import Optional, List, Dict

def process_user(data: Dict[str, Any]) -> Optional[User]: ...
```

### TypeAlias and TypeVar

```python
from typing import Any, TypeAlias, TypeVar

# Explicit type alias
JSON: TypeAlias = dict[str, Any] | list[Any] | str | int | float | bool | None


def parse_json(data: str) -> JSON:
    import json
    return json.loads(data)


# Generic types
T = TypeVar("T")


def first(items: list[T]) -> T | None:
    """Return the first item or None if list is empty."""
    return items[0] if items else None
```

### Self Type (Python 3.11+)

```python
from typing import Self


class Builder:
    def set_name(self, name: str) -> Self:
        self._name = name
        return self

    def set_value(self, value: int) -> Self:
        self._value = value
        return self
```

### Protocol-Based Duck Typing

```python
from typing import Protocol


class Renderable(Protocol):
    def render(self) -> str:
        """Render the object to a string."""


def render_all(items: list[Renderable]) -> str:
    """Render all items that implement the Renderable protocol."""
    return "\n".join(item.render() for item in items)
```

### TypeGuard for Narrowing

```python
from typing import TypeGuard


def is_string_list(val: list[object]) -> TypeGuard[list[str]]:
    return all(isinstance(x, str) for x in val)
```

## Structural Pattern Matching (Python 3.10+)

```python
# Match on type and structure - replaces isinstance chains
def handle_command(command: dict[str, Any]) -> str:
    match command:
        case {"action": "quit"}:
            return "Goodbye!"
        case {"action": "go", "direction": direction}:
            return f"Going {direction}"
        case {"action": action}:
            return f"Unknown action: {action}"
        case _:
            return "Invalid command"


# Match on types
def describe(value: object) -> str:
    match value:
        case int() | float():
            return f"Number: {value}"
        case str():
            return f"String of length {len(value)}"
        case list():
            return f"List with {len(value)} items"
        case _:
            return "Unknown"
```

## Error Handling Patterns

### Specific Exception Handling

```python
import json
from pathlib import Path


def load_config(path: Path) -> Config:
    try:
        return Config.from_json(path.read_text())
    except FileNotFoundError as e:
        raise ConfigError(f"Config file not found: {path}") from e
    except json.JSONDecodeError as e:
        raise ConfigError(f"Invalid JSON in config: {path}") from e
```

### Exception Groups (Python 3.11+)

```python
# Raise multiple exceptions at once
def validate_all(items: list[str]) -> None:
    errors = [ValueError(f"Invalid: {item}") for item in items if not item]
    if errors:
        raise ExceptionGroup("Validation failed", errors)


# Handle exception groups
try:
    validate_all(data)
except* ValueError as eg:
    for err in eg.exceptions:
        print(f"  - {err}")
```

### Custom Exception Hierarchy

```python
class AppError(Exception):
    """Base exception for all application errors."""


class ValidationError(AppError):
    """Raised when input validation fails."""


class NotFoundError(AppError):
    """Raised when a requested resource is not found."""


def get_user(user_id: str) -> User:
    user = db.find_user(user_id)
    if not user:
        raise NotFoundError(f"User not found: {user_id}")
    return user
```

## Pathlib Over `os.path`

Always use `pathlib.Path` for filesystem operations.

```python
from pathlib import Path

# Good: pathlib
def read_config(path: Path) -> str:
    return path.read_text(encoding="utf-8")

config_dir = Path.home() / ".config" / "myapp"
config_dir.mkdir(parents=True, exist_ok=True)

# Find all Python files recursively
py_files = list(Path("src").rglob("*.py"))

# Bad: os.path
import os
config_dir = os.path.join(os.path.expanduser("~"), ".config", "myapp")
```

## Context Managers

### Custom Context Managers

```python
import time
from collections.abc import Generator
from contextlib import contextmanager


@contextmanager
def timer(name: str) -> Generator[None, None, None]:
    """Context manager to time a block of code."""
    start = time.perf_counter()
    yield
    elapsed = time.perf_counter() - start
    print(f"{name} took {elapsed:.4f} seconds")


with timer("data processing"):
    process_large_dataset()
```

### Context Manager Classes

```python
class DatabaseTransaction:
    def __init__(self, connection: Connection) -> None:
        self.connection = connection

    def __enter__(self) -> Self:
        self.connection.begin_transaction()
        return self

    def __exit__(
        self,
        exc_type: type[BaseException] | None,
        exc_val: BaseException | None,
        exc_tb: TracebackType | None,
    ) -> bool:
        if exc_type is None:
            self.connection.commit()
        else:
            self.connection.rollback()
        return False  # Don't suppress exceptions


with DatabaseTransaction(conn):
    user = conn.create_user(user_data)
    conn.create_profile(user.id, profile_data)
```

## Comprehensions and Generators

```python
from collections.abc import Iterable, Iterator

# Good: List comprehension for simple transformations
names = [user.name for user in users if user.is_active]

# Good: Generator for lazy evaluation (no intermediate list)
total = sum(x * x for x in range(1_000_000))

# Expand complex comprehensions into functions
def positive_doubles(items: Iterable[int]) -> list[int]:
    return [x * 2 for x in items if x > 0 and x % 2 == 0]


# Generator function for large files
def read_lines(path: Path) -> Iterator[str]:
    """Read a large file line by line."""
    with path.open() as f:
        for line in f:
            yield line.strip()
```

## Dataclasses and Named Tuples

### Dataclasses

```python
from dataclasses import dataclass, field
from datetime import datetime


@dataclass
class User:
    """User entity with automatic __init__, __repr__, and __eq__."""

    id: str
    name: str
    email: str
    created_at: datetime = field(default_factory=datetime.now)
    is_active: bool = True

    def __post_init__(self) -> None:
        if "@" not in self.email:
            raise ValueError(f"Invalid email: {self.email}")


# Frozen dataclass (immutable, hashable)
@dataclass(frozen=True)
class Point:
    x: float
    y: float

    def distance(self, other: Self) -> float:
        return ((self.x - other.x) ** 2 + (self.y - other.y) ** 2) ** 0.5
```

### Named Tuples

```python
from typing import NamedTuple


class Coordinate(NamedTuple):
    """Immutable geographic coordinate."""

    latitude: float
    longitude: float
    altitude: float = 0.0
```

## Decorators

### Function Decorators

```python
import functools
import time
from collections.abc import Callable
from typing import Any


def timer(func: Callable[..., Any]) -> Callable[..., Any]:
    """Decorator to time function execution."""

    @functools.wraps(func)
    def wrapper(*args: Any, **kwargs: Any) -> Any:
        start = time.perf_counter()
        result = func(*args, **kwargs)
        elapsed = time.perf_counter() - start
        print(f"{func.__name__} took {elapsed:.4f}s")
        return result

    return wrapper


@timer
def slow_function() -> None:
    time.sleep(1)
```

### Parameterized Decorators

```python
def retry(times: int, exceptions: tuple[type[Exception], ...] = (Exception,)):
    """Retry a function up to `times` on specified exceptions."""

    def decorator(func: Callable[..., Any]) -> Callable[..., Any]:
        @functools.wraps(func)
        def wrapper(*args: Any, **kwargs: Any) -> Any:
            for attempt in range(times):
                try:
                    return func(*args, **kwargs)
                except exceptions:
                    if attempt == times - 1:
                        raise
            return None

        return wrapper

    return decorator


@retry(times=3, exceptions=(TimeoutError, ConnectionError))
def fetch_data(url: str) -> bytes: ...
```

## Concurrency Patterns

### Async/Await for I/O-Bound Tasks (Preferred)

```python
import asyncio
from collections.abc import AsyncIterator


async def fetch_one(session: aiohttp.ClientSession, url: str) -> str:
    """Fetch a single URL asynchronously."""
    async with session.get(url) as response:
        response.raise_for_status()
        return await response.text()


async def fetch_all(urls: list[str]) -> dict[str, str | Exception]:
    """Fetch multiple URLs concurrently."""
    async with aiohttp.ClientSession() as session:
        tasks = [fetch_one(session, url) for url in urls]
        results = await asyncio.gather(*tasks, return_exceptions=True)
    return dict(zip(urls, results))


# Async generators
async def paginate(client: APIClient, endpoint: str) -> AsyncIterator[dict]:
    page = 1
    while True:
        data = await client.get(endpoint, params={"page": page})
        if not data:
            break
        yield data
        page += 1
```

### ThreadPoolExecutor for I/O Without async

```python
import concurrent.futures


def fetch_all_sync(urls: list[str]) -> dict[str, str]:
    """Fetch multiple URLs concurrently using threads."""
    with concurrent.futures.ThreadPoolExecutor(max_workers=10) as executor:
        future_to_url = {executor.submit(fetch_url, url): url for url in urls}
        results: dict[str, str] = {}
        for future in concurrent.futures.as_completed(future_to_url):
            url = future_to_url[future]
            try:
                results[url] = future.result()
            except Exception as e:  # noqa: BLE001
                results[url] = f"Error: {e}"
    return results
```

### ProcessPoolExecutor for CPU-Bound Tasks

```python
def process_all(datasets: list[list[int]]) -> list[int]:
    """Process multiple datasets using multiple processes."""
    with concurrent.futures.ProcessPoolExecutor() as executor:
        return list(executor.map(process_data, datasets))
```

## Package Organization

### Standard Project Layout (src layout)

```
myproject/
├── src/
│   └── mypackage/
│       ├── __init__.py
│       ├── py.typed          # PEP 561 marker
│       ├── main.py
│       ├── api/
│       │   ├── __init__.py
│       │   └── routes.py
│       ├── models/
│       │   ├── __init__.py
│       │   └── user.py
│       └── utils/
│           ├── __init__.py
│           └── helpers.py
├── tests/
│   ├── conftest.py
│   ├── test_api.py
│   └── test_models.py
├── pyproject.toml
└── .gitignore
```

### Import Conventions

Ruff (`I` rules) enforces import order automatically: stdlib → third-party → local.

```python
# stdlib
import json
from pathlib import Path

# third-party
import httpx
from fastapi import FastAPI

# local
from mypackage.models import User
from mypackage.utils import format_name
```

### `__init__.py` for Package Exports

```python
# mypackage/__init__.py
"""mypackage - A sample Python package."""

__version__ = "1.0.0"

from mypackage.models import Post, User
from mypackage.utils import format_name

__all__ = ["User", "Post", "format_name"]
```

## Memory and Performance

### Use `__slots__` or Frozen Dataclasses

```python
# __slots__ for plain classes - reduces per-instance memory
class Point:
    __slots__ = ("x", "y")

    def __init__(self, x: float, y: float) -> None:
        self.x = x
        self.y = y


# Frozen dataclass - immutable + slot optimisation
@dataclass(frozen=True, slots=True)
class Vector:
    x: float
    y: float
```

### Avoid String Concatenation in Loops

```python
# Bad: O(n²)
result = ""
for item in items:
    result += str(item)

# Good: O(n)
result = "".join(str(item) for item in items)
```

## Tooling

### Essential Commands

```bash
# Format + lint (replaces black + isort + flake8)
ruff format .
ruff check .
ruff check --fix .

# Type checking
mypy src/

# Testing with coverage
pytest --cov=mypackage --cov-report=term-missing

# Security scanning
bandit -r src/

# Dependency auditing
pip-audit
```

## Quick Reference: Python 3.11+ Idioms

| Idiom | Notes |
|-------|-------|
| `X \| Y` union syntax | Replaces `Optional[X]` / `Union[X, Y]` |
| `match` statement | Structural pattern matching (3.10+) |
| `ExceptionGroup` / `except*` | Multi-error handling (3.11+) |
| `Self` type | Return type for fluent APIs (3.11+) |
| `@dataclass(slots=True)` | Auto `__slots__` (3.10+) |
| `pathlib.Path` | All filesystem operations |
| `asyncio.gather` | Concurrent async I/O |
| Generator expressions | Lazy evaluation, large datasets |
| `ruff format` + `ruff check` | Single tool for format + lint |

## Anti-Patterns to Avoid

```python
# Bad: Mutable default argument
def append_to(item: str, items: list[str] = []) -> list[str]:  # noqa: B006
    items.append(item)
    return items

# Good: Use None sentinel
def append_to(item: str, items: list[str] | None = None) -> list[str]:
    if items is None:
        items = []
    items.append(item)
    return items


# Bad: type() comparison (doesn't respect subclasses)
if type(obj) == list:
    process(obj)

# Good: isinstance
if isinstance(obj, list):
    process(obj)


# Bad: == None
if value == None:
    process()

# Good: is None
if value is None:
    process()


# Bad: wildcard imports
from os.path import *

# Good: explicit
from os.path import exists, join


# Bad: bare except
try:
    risky()
except:
    pass

# Good: specific exception
try:
    risky()
except SpecificError as e:
    logger.error("Operation failed: %s", e)


# Bad: legacy typing imports (Python < 3.9)
from typing import Dict, List, Optional, Tuple, Union

# Good: built-ins + | syntax
def fn(items: list[str], mapping: dict[str, int]) -> tuple[str, ...] | None: ...
```

**Remember**: Python code should be readable, explicit, and follow the principle of least surprise. When in doubt, prioritize clarity over cleverness.
