---
description: "Documentation standards and conventions for Python projects"
applyTo: "**/*.py,**/*.pyi,**/*.md,**/*.rst"
---

# Documentation Standards

## High-Level Principles

- Documentation is part of the codebase: it must be maintained, reviewed, and tested alongside code.
- Write documentation for two audiences: **developers** (code-level docs) and **users/consumers** (API docs, guides).
- Prefer self-documenting code (clear names, types, small functions) supplemented by documentation for intent and context.

## Docstrings

### General Rules

- All public modules, classes, functions, and methods **must** have docstrings.
- Use **Google style** docstrings as the default convention (compatible with Sphinx, MkDocs, and most LLM agents).
- Write docstrings in imperative mood ("Return the result" not "Returns the result").
- Keep the first line as a concise summary (max 79 characters), followed by a blank line and detailed description if needed.

### Google Style Docstring Format

```python
def calculate_discount(
    price: float,
    discount_rate: float,
    *,
    min_price: float = 0.0,
) -> float:
    """Calculate the discounted price for a given item.

    Apply the discount rate to the original price, ensuring the result
    does not fall below the minimum price threshold.

    Args:
        price: The original price of the item. Must be non-negative.
        discount_rate: The discount rate as a decimal (e.g., 0.15 for 15%).
            Must be between 0.0 and 1.0.
        min_price: The minimum allowed price after discount. Defaults to 0.0.

    Returns:
        The discounted price, guaranteed to be >= min_price.

    Raises:
        ValueError: If price is negative or discount_rate is out of range.

    Examples:
        >>> calculate_discount(100.0, 0.2)
        80.0
        >>> calculate_discount(50.0, 0.9, min_price=10.0)
        10.0
    """
    ...
```

### Docstring Sections (Google Style)

| Section | Usage |
|---------|-------|
| `Args:` | Describe each parameter (name, type implicit from annotation, description) |
| `Returns:` | Describe the return value and its semantics |
| `Yields:` | For generators — describe what is yielded |
| `Raises:` | List exceptions the function may raise and under what conditions |
| `Examples:` | Provide usage examples (preferably doctestable) |
| `Notes:` | Implementation notes, algorithm references, caveats |
| `See Also:` | Cross-references to related functions or classes |
| `Attributes:` | For classes — describe public instance attributes |

### Class Docstrings

```python
class TaskQueue:
    """A thread-safe queue for managing background tasks.

    Provides methods to enqueue, dequeue, and inspect tasks. Tasks are
    processed in FIFO order with optional priority support.

    Attributes:
        max_size: Maximum number of tasks the queue can hold.
        timeout: Default timeout in seconds for blocking operations.

    Examples:
        >>> queue = TaskQueue(max_size=100)
        >>> queue.enqueue(Task("process_data"))
        >>> task = queue.dequeue()
    """

    def __init__(self, max_size: int = 0, timeout: float = 30.0) -> None:
        """Initialize the task queue.

        Args:
            max_size: Maximum queue capacity. 0 means unlimited.
            timeout: Default timeout for blocking operations in seconds.
        """
        ...
```

### Module Docstrings

```python
"""HTTP client utilities for interacting with external APIs.

This module provides a high-level HTTP client with retry logic,
timeout management, and structured error handling. It wraps the
`requests` library with project-specific conventions.

Typical usage:
    from myproject.http import Client

    client = Client(base_url="https://api.example.com")
    response = client.get("/users", params={"active": True})
"""
```

## Inline Comments

- Use inline comments sparingly — only to explain **why**, not **what**.
- Place comments on their own line above the code they explain.
- Do not comment obvious code.
- Use `# TODO:`, `# FIXME:`, `# HACK:`, `# NOTE:` prefixes for actionable comments.

```python
# Good: explains intent
# Retry with exponential backoff because the upstream API
# is rate-limited and returns 429 during peak hours.
for attempt in range(max_retries):
    ...

# Bad: restates the code
# Increment counter by 1
counter += 1
```

## README Files

Every project and significant sub-package should have a `README.md` containing:

1. **Title and description**: What the project/module does.
2. **Installation**: How to set up the development environment.
3. **Quick start**: Minimal example to get started.
4. **Configuration**: Required environment variables and settings.
5. **Usage**: Key features and API overview.
6. **Development**: How to run tests, lint, and contribute.
7. **Architecture**: High-level design (optional, for larger projects).

## Changelog

- Maintain a `CHANGELOG.md` following [Keep a Changelog](https://keepachangelog.com/) format.
- Group changes by: `Added`, `Changed`, `Deprecated`, `Removed`, `Fixed`, `Security`.
- Link each version to its Git tag or comparison URL.

```markdown
## [1.2.0] - 2026-03-15

### Added
- Support for batch processing in `DataPipeline`.
- New `--dry-run` flag for CLI commands.

### Fixed
- Race condition in `TaskQueue.dequeue()` under high concurrency.

### Deprecated
- `process_single()` — use `process_batch()` with a single-item list instead.
```

## API Documentation

- Use **Sphinx** with `autodoc` or **MkDocs** with `mkdocstrings` for generating API docs from docstrings.
- Ensure all public API entries are included in the documentation index.
- Include code examples in API documentation for key functions and classes.
- Keep examples up to date; consider using `doctest` or `pytest --doctest-modules` to validate them.

## Architecture Decision Records (ADRs)

For significant design decisions, create an ADR in `docs/adr/`:

```markdown
# ADR-001: Use Pydantic for Data Validation

## Status
Accepted

## Context
We need a consistent way to validate external input across all API endpoints.

## Decision
Use Pydantic v2 for all data validation and serialization.

## Consequences
- All input models inherit from `pydantic.BaseModel`.
- Serialization to/from JSON is handled by Pydantic.
- Dependency on Pydantic v2+ is required.
```

## Diagrams

- Use **Mermaid** diagrams in Markdown for architecture and flow documentation.
- Keep diagrams close to the code they describe.
- For complex systems, include both high-level and component-level diagrams.

## Notes for LLM Agent Behavior

- Always generate docstrings for new functions, classes, and modules.
- Follow the Google docstring style unless the project explicitly uses another convention.
- Include `Args`, `Returns`, and `Raises` sections in all function docstrings.
- Add usage examples in docstrings when the function's behavior is non-obvious.
- When modifying existing code, update the corresponding docstrings.
- Do not generate redundant documentation that restates obvious type annotations.
