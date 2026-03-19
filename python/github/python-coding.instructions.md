---
description: "Python coding conventions and best practices"
applyTo: "**/*.py,**/*.pyi,**/*.pyw"
---

# Python Coding Guidelines

## High-Level Principles

- Follow the Zen of Python (`import this`): prefer explicit over implicit, simple over complex, flat over nested.
- Write code that is readable, predictable, and easy to test.
- Keep patches small and testable: prefer incremental changes that pass all tests and linting.
- Optimize for readability first; optimize for performance only when profiling justifies it.

## Style & Formatting

### General Rules

- Follow **PEP 8** as the baseline style guide.
- Use **`black`** for automatic code formatting (line length: 88 characters by default, or 120 if project convention).
- Use **`isort`** (or `ruff` with isort rules) for import sorting.
- Use **`ruff`** as the primary linter for fast, comprehensive checks.

### Naming Conventions

| Element | Convention | Example |
|---------|-----------|---------|
| Functions / Methods | `snake_case`, verb-first | `calculate_total()`, `send_message()` |
| Variables | `snake_case` | `user_count`, `max_retries` |
| Classes | `PascalCase` | `DataProcessor`, `UserSession` |
| Constants | `UPPER_SNAKE_CASE` | `MAX_RETRIES`, `DEFAULT_TIMEOUT` |
| Modules / Packages | `snake_case`, short | `utils`, `data_access` |
| Private members | Leading underscore | `_internal_method()`, `_cache` |
| Protected members | Leading underscore | `_validate_input()` |
| "Dunder" methods | Double underscore | `__init__()`, `__repr__()` |
| Type variables | `PascalCase`, short | `T`, `KeyType`, `ValueType` |

### Import Organization

Imports should be organized in this order, separated by blank lines:

1. Standard library imports
2. Third-party library imports
3. Local application/project imports

```python
import os
import sys
from pathlib import Path

import requests
from pydantic import BaseModel

from myproject.core import Engine
from myproject.utils import helpers
```

- Prefer absolute imports over relative imports.
- Avoid wildcard imports (`from module import *`).
- Import specific names rather than entire modules when only a few are needed.

## Type Hints & Annotations

- **Always** annotate public function signatures (parameters and return types).
- Use `from __future__ import annotations` for modern annotation syntax in Python 3.9+.
- Use built-in generic types (`list[str]`, `dict[str, int]`) instead of `typing.List`, `typing.Dict` (Python 3.9+).
- Use `X | None` instead of `Optional[X]` (Python 3.10+).
- Use `TypeAlias` for complex type definitions.
- Use `Protocol` for structural subtyping instead of ABCs when appropriate.
- Use `TypedDict` for dictionaries with known key structures.

```python
from __future__ import annotations

from typing import Protocol, TypeAlias

Coordinates: TypeAlias = tuple[float, float]

class Serializable(Protocol):
    def to_dict(self) -> dict[str, any]: ...

def process_items(items: list[str], *, timeout: float = 30.0) -> dict[str, int]:
    """Process a list of items and return counts."""
    ...
```

## Functions & Methods

- Keep functions short and focused (single responsibility).
- Prefer keyword arguments for functions with more than 2-3 parameters.
- Use `*` to force keyword-only arguments when clarity matters.
- Limit function arguments; if a function takes many arguments, consider grouping into a dataclass or typed dict.
- Return early to reduce nesting.
- Avoid mutable default arguments (`def f(items=[])` — use `None` and initialize inside).

```python
# Good: keyword-only, early return, no mutable default
def fetch_data(
    url: str,
    *,
    timeout: float = 30.0,
    retries: int = 3,
    headers: dict[str, str] | None = None,
) -> Response:
    if not url:
        raise ValueError("URL must not be empty")

    headers = headers or {}
    ...
```

## Classes & Data Structures

- Use **dataclasses** for simple data containers.
- Use **Pydantic models** for data validation and serialization.
- Use **NamedTuple** for immutable, lightweight records.
- Prefer composition over inheritance.
- Keep class interfaces small; extract mixins or helper functions rather than building large class hierarchies.
- Always implement `__repr__` for custom classes used in debugging.
- Use `@property` for computed attributes with simple logic; avoid side effects in properties.

```python
from dataclasses import dataclass, field
from datetime import datetime

@dataclass
class TaskResult:
    task_id: str
    status: str
    created_at: datetime = field(default_factory=datetime.now)
    errors: list[str] = field(default_factory=list)

    @property
    def is_successful(self) -> bool:
        return self.status == "success" and not self.errors
```

## Error Handling

- **Always** catch specific exceptions; never use bare `except:` or `except Exception:` without re-raising.
- Use custom exception classes for domain-specific errors, inheriting from a project-level base exception.
- Prefer EAFP (Easier to Ask Forgiveness than Permission) over LBYL (Look Before You Leap), but use LBYL when the check is cheap and the exception is expensive.
- Use context managers (`with` statements) for resource management.
- Log exceptions with sufficient context before re-raising or wrapping.

```python
class AppError(Exception):
    """Base exception for the application."""

class NotFoundError(AppError):
    """Raised when a requested resource is not found."""

class ValidationError(AppError):
    """Raised when input validation fails."""

# Good: specific exception, context manager, logging
try:
    with open(config_path) as f:
        config = json.load(f)
except FileNotFoundError:
    logger.error("Configuration file not found: %s", config_path)
    raise
except json.JSONDecodeError as e:
    raise ValidationError(f"Invalid JSON in {config_path}") from e
```

## Logging

- Use the standard `logging` module; never use `print()` for application output in production code.
- Use structured logging (`logging` with `extra` fields, or `structlog`) for production systems.
- Use lazy formatting (`logger.info("Processing %s", item)`) instead of f-strings in log calls.
- Define loggers at module level: `logger = logging.getLogger(__name__)`.
- Use appropriate log levels: `DEBUG` for development detail, `INFO` for operational events, `WARNING` for recoverable issues, `ERROR` for failures, `CRITICAL` for system-wide problems.
- Remove or guard debug-level logs in performance-critical paths.

## Performance & Memory

- **Profile before optimizing**. Use `cProfile`, `line_profiler`, or `py-spy` to identify bottlenecks.
- Prefer generators and iterators over lists for large datasets.
- Use `collections` module types (`defaultdict`, `Counter`, `deque`) when they simplify code.
- Use list/dict/set comprehensions for concise transformations; avoid nested comprehensions deeper than 2 levels.
- Cache expensive computations with `functools.lru_cache` or `functools.cache`.
- Avoid premature optimization; prefer clarity unless profiling identifies hot code.

## Concurrency & Async

- Use `asyncio` for I/O-bound concurrency.
- Use `concurrent.futures.ThreadPoolExecutor` for blocking I/O mixed with sync code.
- Use `concurrent.futures.ProcessPoolExecutor` for CPU-bound parallelism.
- Avoid global mutable state in concurrent code.
- Use `asyncio.TaskGroup` (Python 3.11+) for structured concurrency.

## Security

> For comprehensive security guidelines, see [security.instructions.md](security.instructions.md).

- **Never** hardcode secrets, API keys, or credentials in source code — use environment variables or a secrets manager.
- **Validate and sanitize all external inputs** at the boundary using Pydantic models or explicit checks.
- Use `secrets` module for cryptographic randomness — **never** `random` for security-sensitive values.
- **Never** use `eval()`, `exec()`, `pickle.loads()`, or `yaml.load()` on untrusted data.
- **Always** use parameterized queries for database access — never string formatting or concatenation for SQL.
- Prevent path traversal: resolve paths and verify they remain within allowed directories.
- Use `argon2` or `bcrypt` for password hashing — never MD5, SHA-1, or plain SHA-256.
- Set security headers (CSP, HSTS, X-Content-Type-Options) on all HTTP responses.
- Keep dependencies up to date; run `pip audit` or `uv pip audit` regularly and enable Dependabot.
- Implement rate limiting on public-facing endpoints to prevent abuse.
- Log security events (auth failures, input validation errors) but **never** log passwords, tokens, or PII.

## Platform & Compatibility

- Specify minimum Python version in `pyproject.toml` (`requires-python`).
- Use `pathlib.Path` for file system operations instead of `os.path`.
- Use `os.environ` or `python-dotenv` for environment configuration.
- Guard platform-specific code with `sys.platform` checks when necessary.

## Notes for LLM Agent Behavior

- Prioritize suggestions that align with the existing code style and project conventions.
- Propose minimal, test-first edits. Prefer suggestions that are reversible and safe.
- When suggesting API changes, include migration steps and update all callers.
- Always include type hints in suggested code.
- Generate code compatible with the project's minimum Python version.
- Do not suggest running the application unless explicitly asked; provide code changes for the developer to test.
- **Never** generate code that uses `eval()`, `exec()`, `pickle.loads()`, or `yaml.load()` — use safe alternatives.
- **Always** use parameterized queries when generating database code.
- **Always** use `secrets` instead of `random` for tokens, keys, and nonces.
- When generating API endpoints, include input validation, authentication, and error handling that does not leak internals.
- Flag any hardcoded credentials found in code and recommend environment variables or a secrets manager.
