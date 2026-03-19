---
description: "Comprehensive guidelines for AI coding agents in Python projects"
applyTo: "**/*.py,**/*.pyi,**/*.pyw"
---

# Python Copilot Instructions

## Quick Reference

- **Style**: Follow PEP 8; use `black` for formatting, `isort` for imports.
- **Type Hints**: Use type annotations on all public functions and methods.
- **Naming**: `snake_case` for functions/variables, `PascalCase` for classes, `UPPER_SNAKE_CASE` for constants.
- **Error Handling**: Use specific exceptions; avoid bare `except`.
- **Security**: Validate all inputs, never hardcode secrets, use parameterized queries. See security instructions.
- **Testing**: Write tests for all new functionality using `pytest`.
- **Commit Messages**: Follow the Conventional Commits specification (see below).

## Project Architecture

- **Typical Structure**:
  - `src/` or top-level package: Core application logic, organized by domain.
  - `tests/`: Unit and integration tests mirroring the `src/` structure.
  - `docs/`: Documentation (Sphinx, MkDocs, or plain Markdown).
  - `scripts/`: Utility and automation scripts.
  - `config/` or root config files: Configuration (`pyproject.toml`, `.env`, etc.).
- **Service Boundaries**:
  - Communication between components uses well-defined interfaces (protocols, ABCs).
  - External integrations are isolated behind adapter layers.
- **Data Flows**:
  - Data is managed through typed dataclasses, Pydantic models, or typed dictionaries.
  - I/O operations are separated from business logic.

## Developer Workflows

- **Building / Installing**:
  - Use `pyproject.toml` as the single source for project metadata and build configuration.
  - Use virtual environments (`venv`, `uv`, or `conda`) for dependency isolation.
- **Testing**:
  - Run tests with `pytest`. Use `pytest-cov` for coverage reports.
  - Refer to the testing instructions for detailed patterns and conventions.
- **Linting & Formatting**:
  - Use `ruff` (or `flake8` + `black` + `isort`) for consistent code quality.
  - Use `mypy` or `pyright` for static type checking.

## Commit Message Template

```
<type>(<scope>): <short summary in imperative mood, max 72 chars>

<body: what changed and why, wrapped at 72 characters>

Tests:
- List tests performed or added.

Closes: #<issue-number>
```

### Conventional Commit Types

| Type | Usage |
|------|-------|
| `feat` | New feature |
| `fix` | Bug fix |
| `docs` | Documentation changes |
| `style` | Formatting, no logic change |
| `refactor` | Code restructuring, no behavior change |
| `test` | Adding or updating tests |
| `chore` | Build, CI, tooling changes |
| `perf` | Performance improvements |
| `security` | Security fixes or improvements |

## References

- **Python Coding Guidelines**: [python-coding.instructions.md](python-coding.instructions.md)
- **Documentation Standards**: [documentation.instructions.md](documentation.instructions.md)
- **Testing Best Practices**: [testing.instructions.md](testing.instructions.md)
- **Project Structure**: [project-structure.instructions.md](project-structure.instructions.md)
- **Application Security**: [security.instructions.md](security.instructions.md)

## Feedback

If any section is unclear or incomplete, please provide feedback to improve this document.
