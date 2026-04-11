---
name: python-patterns
description: Pythonic idioms, type hints, and best practices for building robust, efficient, and maintainable Python applications.
origin: ECC
---

# Python Development Patterns

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

## Logging

Always use the built-in `logging` module - never `print()` for application output. Configure handlers to write to both console and a log file.

```python
import logging
import sys

# Configure basic logging to file and stdout
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler("app.log"),
        logging.StreamHandler(sys.stdout)
    ]
)
logging.info('Message goes to both file and console')
```

Use module-level loggers in library code (not the root logger):

```python
import logging

logger = logging.getLogger(__name__)

def process_user(user_id: str) -> None:
    logger.info("Processing user: %s", user_id)
    try:
        result = do_work(user_id)
        logger.debug("Result: %s", result)
    except Exception:
        logger.exception("Failed to process user: %s", user_id)
        raise
```

| Level | Use for |
|-------|---------|
| `DEBUG` | Detailed diagnostic info |
| `INFO` | Normal operation events |
| `WARNING` | Unexpected but recoverable |
| `ERROR` | Failure in a specific operation |
| `CRITICAL` | Application-level fatal error |

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

## Dataclasses

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


## Memory and Performance

### Frozen Dataclasses

```python
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

## Quick Reference: Python 3.11+ Idioms

| Idiom | Notes |
|-------|-------|
| `X \| Y` union syntax | Replaces `Optional[X]` / `Union[X, Y]` |
| `ExceptionGroup` / `except*` | Multi-error handling (3.11+) |
| `@dataclass(slots=True)` | Auto `__slots__` (3.10+) |
| `pathlib.Path` | All filesystem operations |
| `asyncio.gather` | Concurrent async I/O |
| Generator expressions | Lazy evaluation, large datasets |

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
from math import *

# Good: explicit
from math import sqrt, pi


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
