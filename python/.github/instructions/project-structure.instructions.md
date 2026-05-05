---
description: "Project structure, packaging, and dependency management conventions for Python"
applyTo: "**/*.py,**/*.toml,**/*.cfg,**/*.ini"
---

# Project Structure & Dependency Management

## High-Level Principles

- Use a consistent, well-known project layout across all repositories.
- `pyproject.toml` is the single source of truth for project metadata, dependencies, and tool configuration.
- Isolate dependencies with virtual environments; never install packages globally.
- Pin dependencies for reproducible builds; use ranges for libraries.

## Standard Project Layout

### Application Layout (src layout)

```
my-project/
├── .github/
│   ├── copilot-instructions.md       # AI agent instructions
│   └── workflows/
│       ├── ci.yml                     # CI pipeline
│       └── release.yml                # Release automation
├── src/
│   └── myproject/
│       ├── __init__.py
│       ├── py.typed                   # PEP 561 marker for type stubs
│       ├── core/
│       │   ├── __init__.py
│       │   ├── engine.py
│       │   └── models.py
│       ├── api/
│       │   ├── __init__.py
│       │   ├── routes.py
│       │   └── schemas.py
│       ├── services/
│       │   ├── __init__.py
│       │   └── data_service.py
│       └── utils/
│           ├── __init__.py
│           └── helpers.py
├── tests/
│   ├── conftest.py
│   ├── unit/
│   │   └── ...
│   └── integration/
│       └── ...
├── docs/
│   ├── index.md
│   ├── adr/
│   │   └── 001-use-pydantic.md
│   └── api/
│       └── reference.md
├── scripts/
│   ├── setup_dev.sh
│   └── run_migrations.py
├── .env.example
├── .gitignore
├── .pre-commit-config.yaml
├── CHANGELOG.md
├── LICENSE
├── Makefile
├── README.md
└── pyproject.toml
```

### Key Directory Roles

| Directory | Purpose |
|-----------|---------|
| `src/<package>/` | Application source code, organized by domain |
| `tests/` | All tests, mirroring src structure |
| `docs/` | Project documentation |
| `scripts/` | Development and automation scripts |
| `.github/` | CI/CD workflows and AI agent instructions |

### Why src Layout?

- Prevents accidental imports from the working directory during development.
- Forces installation of the package for testing, catching packaging issues early.
- Is the recommended layout by PyPA and major tools.

## pyproject.toml Configuration

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "myproject"
version = "1.0.0"
description = "Brief description of the project."
readme = "README.md"
license = { text = "MIT" }
requires-python = ">=3.11"
authors = [
    { name = "Author Name", email = "author@example.com" },
]
classifiers = [
    "Programming Language :: Python :: 3",
    "Programming Language :: Python :: 3.11",
    "Programming Language :: Python :: 3.12",
]
dependencies = [
    "pydantic>=2.0,<3.0",
    "httpx>=0.25.0",
    "structlog>=23.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0",
    "pytest-cov>=5.0",
    "pytest-asyncio>=0.23",
    "pytest-mock>=3.12",
    "mypy>=1.8",
    "ruff>=0.3",
    "pre-commit>=3.6",
    "bandit>=1.7",
    "pip-audit>=2.7",
]
docs = [
    "mkdocs>=1.5",
    "mkdocstrings[python]>=0.24",
]

[project.scripts]
myproject = "myproject.cli:main"

# --- Tool Configuration ---

[tool.ruff]
target-version = "py311"
line-length = 88
src = ["src", "tests"]

[tool.ruff.lint]
select = [
    "E",     # pycodestyle errors
    "W",     # pycodestyle warnings
    "F",     # pyflakes
    "I",     # isort
    "N",     # pep8-naming
    "UP",    # pyupgrade
    "B",     # flake8-bugbear
    "SIM",   # flake8-simplify
    "TCH",   # flake8-type-checking
    "RUF",   # ruff-specific rules
    "S",     # flake8-bandit (security checks)
]

[tool.ruff.lint.isort]
known-first-party = ["myproject"]

[tool.mypy]
python_version = "3.11"
strict = true
warn_return_any = true
warn_unused_configs = true
disallow_untyped_defs = true

[[tool.mypy.overrides]]
module = "tests.*"
disallow_untyped_defs = false

[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = ["--strict-markers", "--strict-config", "-ra"]
markers = [
    "integration: marks integration tests",
    "e2e: marks end-to-end tests",
    "slow: marks slow tests",
]
```

## Dependency Management

### For Applications (Pinned Dependencies)

- Pin exact versions in a lock file for reproducible deployments.
- Use **`uv`** (recommended), **`pip-compile`** (pip-tools), or **`poetry.lock`** for lock files.
- Keep `pyproject.toml` with compatible ranges; generate lock file from it.

```bash
# Using uv
uv lock
uv sync

# Using pip-tools
pip-compile pyproject.toml -o requirements.lock
pip install -r requirements.lock
```

### For Libraries (Version Ranges)

- Use compatible version ranges in `dependencies`.
- Be as permissive as possible to avoid conflicts for consumers.
- Test against multiple Python versions and dependency versions in CI.

```toml
dependencies = [
    "pydantic>=2.0,<3.0",    # Compatible range
    "httpx>=0.25.0",          # Minimum version
]
```

### Virtual Environments

- **Always** use a virtual environment for development.
- Prefer **`uv`** for fast environment and dependency management.
- Document the setup process in `README.md` and/or a `Makefile`.

```makefile
# Makefile
.PHONY: setup test lint format typecheck

setup:
	uv venv
	uv sync --group dev

test:
	uv run pytest --cov

lint:
	uv run ruff check src tests

format:
	uv run ruff format src tests

typecheck:
	uv run mypy src
```

## CI/CD Conventions

### GitHub Actions Workflow

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ["3.11", "3.12", "3.13"]
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v4
      - run: uv sync --group dev
      - run: uv run ruff check src tests
      - run: uv run mypy src
      - run: uv run pytest --cov --cov-report=xml
      - name: Security audit
        run: uv run pip-audit
      - name: Bandit security linter
        run: uv run bandit -r src/ -c pyproject.toml
```

## Pre-Commit Hooks

Use `.pre-commit-config.yaml` for automated code quality checks:

```yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.3.0
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format
  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.8.0
    hooks:
      - id: mypy
        additional_dependencies: [pydantic>=2.0]
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-toml
      - id: check-added-large-files
  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.4.0
    hooks:
      - id: detect-secrets
        args: ['--baseline', '.secrets.baseline']
```

## Configuration Management

- Use **environment variables** for deployment-specific configuration.
- Use **`.env` files** for local development (with `.env.example` committed).
- Use **Pydantic Settings** (`pydantic-settings`) for typed configuration with validation.
- Never commit secrets or credentials to the repository.
- Use `SecretStr` from Pydantic for sensitive fields to prevent accidental logging.
- Enable `detect-secrets` pre-commit hook to catch accidental secret commits.

```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    database_url: str
    api_key: str
    debug: bool = False
    log_level: str = "INFO"

    model_config = {"env_prefix": "APP_", "env_file": ".env"}
```

## Notes for LLM Agent Behavior

- When creating new projects, follow the src layout.
- Always generate a `pyproject.toml` with build system, dependencies, and tool configuration.
- Include `py.typed` marker for packages that export type information.
- Suggest `Makefile` or equivalent task runner for common development commands.
- Use `uv` as the default recommendation for dependency management.
- Do not suggest global package installations.
- Always include `bandit` and `pip-audit` in dev dependencies for new projects.
- Include security scanning steps (dependency audit, Bandit, detect-secrets) in CI workflows.
- Never generate code that commits secrets or credentials; use environment variables and `SecretStr`.
