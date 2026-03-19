---
description: "Testing patterns and best practices for Python projects"
applyTo: "**/*.py,**/test_*.py,**/*_test.py,**/conftest.py"
---

# Testing Best Practices

## High-Level Principles

- Every new feature or bug fix must include corresponding tests.
- Tests are first-class code: readable, maintainable, and well-organized.
- Aim for high coverage (>80%) but prioritize meaningful tests over coverage metrics.
- Tests should be fast, isolated, deterministic, and independent of execution order.

## Testing Framework

- Use **`pytest`** as the primary testing framework.
- Use **`pytest-cov`** for coverage reporting.
- Use **`pytest-xdist`** for parallel test execution when test suites grow.
- Use **`pytest-mock`** (wraps `unittest.mock`) for mocking.
- Use **`hypothesis`** for property-based testing when testing complex logic.
- Use **`pytest-asyncio`** for testing async code.

## Test Organization

### Directory Structure

```
project/
├── src/
│   └── myproject/
│       ├── core/
│       │   ├── __init__.py
│       │   └── engine.py
│       └── utils/
│           ├── __init__.py
│           └── helpers.py
├── tests/
│   ├── conftest.py              # Shared fixtures
│   ├── unit/
│   │   ├── conftest.py          # Unit test fixtures
│   │   ├── core/
│   │   │   └── test_engine.py
│   │   └── utils/
│   │       └── test_helpers.py
│   ├── integration/
│   │   ├── conftest.py          # Integration test fixtures
│   │   └── test_api.py
│   └── e2e/
│       └── test_workflows.py
```

### Naming Conventions

| Element | Convention | Example |
|---------|-----------|---------|
| Test files | `test_<module>.py` | `test_engine.py` |
| Test classes | `Test<Feature>` | `TestEngine`, `TestUserAuth` |
| Test functions | `test_<behavior>` | `test_calculate_total_with_discount` |
| Fixtures | Descriptive `snake_case` | `sample_user`, `db_session` |
| Conftest files | `conftest.py` | Shared fixtures per directory |

### Test Naming Pattern

Use descriptive names that express **what is being tested** and **under what condition**:

```python
# Pattern: test_<action>_<condition>_<expected_result>
def test_calculate_discount_with_zero_rate_returns_original_price(): ...
def test_create_user_with_duplicate_email_raises_conflict(): ...
def test_fetch_data_when_server_unavailable_retries_three_times(): ...
```

## Writing Tests

### Basic Test Structure (Arrange-Act-Assert)

```python
def test_calculate_total_applies_tax(sample_items: list[Item]) -> None:
    """Verify that calculate_total includes the correct tax amount."""
    # Arrange
    tax_rate = 0.21
    expected_total = 121.0

    # Act
    result = calculate_total(sample_items, tax_rate=tax_rate)

    # Assert
    assert result == pytest.approx(expected_total)
```

### Fixtures

```python
# conftest.py
import pytest
from myproject.models import User, Database

@pytest.fixture
def sample_user() -> User:
    """Create a sample user for testing."""
    return User(name="Alice", email="alice@example.com", role="admin")

@pytest.fixture
def db_session(tmp_path: Path) -> Generator[Database, None, None]:
    """Provide a temporary database session that is cleaned up after the test."""
    db = Database(path=tmp_path / "test.db")
    db.initialize()
    yield db
    db.close()
```

- Use fixtures for setup and teardown; avoid `setUp` / `tearDown` methods.
- Keep fixtures focused: one fixture per concern.
- Use `yield` fixtures for resource cleanup (replaces `tearDown`).
- Scope fixtures appropriately: `function` (default), `class`, `module`, `session`.
- Place shared fixtures in `conftest.py` at the appropriate directory level.

### Parametrized Tests

```python
@pytest.mark.parametrize(
    "price, discount_rate, expected",
    [
        (100.0, 0.0, 100.0),
        (100.0, 0.1, 90.0),
        (100.0, 0.5, 50.0),
        (100.0, 1.0, 0.0),
        (0.0, 0.5, 0.0),
    ],
    ids=[
        "no_discount",
        "ten_percent",
        "half_price",
        "full_discount",
        "zero_price",
    ],
)
def test_calculate_discount(price: float, discount_rate: float, expected: float) -> None:
    assert calculate_discount(price, discount_rate) == pytest.approx(expected)
```

### Mocking

```python
from unittest.mock import patch, MagicMock

def test_send_notification_calls_email_service(sample_user: User) -> None:
    """Verify that send_notification delegates to the email service."""
    with patch("myproject.notifications.email_service") as mock_email:
        mock_email.send.return_value = True

        result = send_notification(sample_user, message="Hello")

        mock_email.send.assert_called_once_with(
            to=sample_user.email,
            subject="Notification",
            body="Hello",
        )
        assert result is True
```

- Mock at the boundary: mock external services, I/O, and time — not internal logic.
- Use `spec=True` or `autospec=True` to ensure mocks match the real interface.
- Prefer dependency injection over patching when possible.
- Avoid over-mocking: if you mock too many things, the test may not verify real behavior.

### Testing Exceptions

```python
def test_create_user_with_invalid_email_raises_validation_error() -> None:
    with pytest.raises(ValidationError, match="Invalid email format"):
        create_user(name="Bob", email="not-an-email")
```

### Testing Async Code

```python
import pytest

@pytest.mark.asyncio
async def test_fetch_data_returns_expected_result(mock_api: MockAPI) -> None:
    result = await fetch_data(url="https://api.example.com/data")
    assert result.status == "ok"
    assert len(result.items) > 0
```

## Test Categories

### Unit Tests

- Test individual functions and methods in isolation.
- Must be fast (<100ms per test) and have no external dependencies.
- Mock all I/O, network, and database calls.
- Target: cover all branches and edge cases of business logic.

### Integration Tests

- Test interactions between components (e.g., service + database, API + auth).
- May use test databases, Docker containers, or test servers.
- Mark with `@pytest.mark.integration` for selective execution.

### End-to-End (E2E) Tests

- Test complete workflows from user perspective.
- Run against a fully configured environment (staging or local).
- Mark with `@pytest.mark.e2e` and exclude from default `pytest` runs.

### Property-Based Tests (Hypothesis)

```python
from hypothesis import given, strategies as st

@given(
    price=st.floats(min_value=0.0, max_value=10000.0),
    rate=st.floats(min_value=0.0, max_value=1.0),
)
def test_discount_never_exceeds_original_price(price: float, rate: float) -> None:
    result = calculate_discount(price, rate)
    assert 0 <= result <= price
```

### Security Tests

Security tests verify that the application correctly defends against common attack vectors. Mark them with `@pytest.mark.security` for selective execution.

```python
import pytest

@pytest.mark.security
class TestInputValidation:
    """Verify that malicious inputs are rejected."""

    @pytest.mark.parametrize(
        "malicious_input",
        [
            "<script>alert('xss')</script>",
            "'; DROP TABLE users; --",
            "../../../etc/passwd",
            "\x00null_byte_injection",
            "A" * 1_000_000,  # Buffer overflow attempt
        ],
        ids=["xss", "sql_injection", "path_traversal", "null_byte", "overflow"],
    )
    def test_user_input_rejects_malicious_values(self, malicious_input: str) -> None:
        with pytest.raises((ValidationError, ValueError)):
            validate_user_input(malicious_input)


@pytest.mark.security
def test_login_rate_limiting(client: TestClient) -> None:
    """Verify that excessive login attempts are blocked."""
    for _ in range(10):
        client.post("/auth/login", json={"email": "a@b.com", "password": "wrong"})
    response = client.post("/auth/login", json={"email": "a@b.com", "password": "wrong"})
    assert response.status_code == 429


@pytest.mark.security
def test_api_returns_generic_error_on_internal_failure(client: TestClient) -> None:
    """Verify that internal errors do not leak sensitive details."""
    response = client.get("/api/v1/trigger-error")
    assert response.status_code == 500
    body = response.json()
    assert "traceback" not in body.get("detail", "").lower()
    assert "password" not in body.get("detail", "").lower()


@pytest.mark.security
def test_auth_required_on_protected_endpoints(client: TestClient) -> None:
    """Verify that protected endpoints reject unauthenticated requests."""
    response = client.get("/api/v1/protected-resource")
    assert response.status_code in (401, 403)
```

#### Security Test Categories

| Category | Focus | Examples |
|----------|-------|----------|
| Input validation | Reject malicious inputs | XSS, SQL injection, path traversal, overflows |
| Authentication | Auth enforcement | Missing tokens, expired tokens, invalid tokens |
| Authorization | Permission enforcement | Role escalation, accessing other users' data |
| Rate limiting | Abuse prevention | Brute-force login, API flooding |
| Error handling | No information leakage | Stack traces, file paths, internal details |
| Crypto | Correct usage | Weak hashing, insecure random, token predictability |
| Headers | Security headers present | CSP, HSTS, X-Content-Type-Options |

## Test Configuration

### `pyproject.toml`

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
python_files = ["test_*.py"]
python_classes = ["Test*"]
python_functions = ["test_*"]
addopts = [
    "--strict-markers",
    "--strict-config",
    "-ra",
]
markers = [
    "integration: marks tests as integration tests",
    "e2e: marks tests as end-to-end tests",
    "slow: marks tests as slow-running",
    "security: marks tests as security tests",
]
filterwarnings = [
    "error",
    "ignore::DeprecationWarning:third_party_lib.*",
]

[tool.coverage.run]
source = ["src"]
branch = true

[tool.coverage.report]
show_missing = true
fail_under = 80
exclude_lines = [
    "pragma: no cover",
    "def __repr__",
    "if TYPE_CHECKING:",
    "raise NotImplementedError",
]
```

## Best Practices Summary

1. **One concept per test**: Each test should verify exactly one behavior.
2. **Descriptive names**: Test names should read like specifications.
3. **No test interdependence**: Tests must not rely on execution order or shared mutable state.
4. **Fast feedback**: Unit tests should run in under a second total; use marks to separate slow tests.
5. **Deterministic**: No flaky tests. Avoid reliance on timing, random data (unless seeded), or external services.
6. **DRY but readable**: Use fixtures and parametrize to reduce duplication, but keep individual tests easy to read.
7. **Test edge cases**: Empty inputs, boundary values, None values, very large inputs, error conditions.
8. **Security tests**: Include tests for input validation, authentication, authorization, rate limiting, and error information leakage.
9. **Maintain tests**: Delete obsolete tests; update tests when behavior changes intentionally.

## Notes for LLM Agent Behavior

- When generating new code, always suggest corresponding test functions.
- Follow the Arrange-Act-Assert pattern in generated tests.
- Use `pytest.raises` for testing exceptions, not try/except.
- Use `pytest.approx` for floating-point comparisons.
- Prefer parametrized tests when testing the same logic with multiple inputs.
- Include descriptive `ids` in parametrized tests for better test output.
- When generating API endpoint code, always suggest corresponding security tests (auth, input validation, error leakage).
- Use parametrized tests with malicious inputs (XSS, SQL injection, path traversal) to verify input validation.
- Include descriptive `ids` in parametrized tests for better test output.
- Always add type hints to test functions and fixtures.
