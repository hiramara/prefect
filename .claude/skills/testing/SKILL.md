# Testing Skill

This skill provides guidance and patterns for writing tests in the Prefect codebase.

## Overview

Prefect uses `pytest` as its primary testing framework. Tests should be written to cover flows, tasks, deployments, and infrastructure components.

## Test Structure

```
tests/
├── conftest.py          # Shared fixtures
├── test_flows.py        # Flow-level tests
├── test_tasks.py        # Task-level tests
├── test_deployments.py  # Deployment tests
└── integrations/        # Integration tests
    └── test_*.py
```

## Key Patterns

### 1. Testing a Flow

```python
import pytest
from prefect import flow, task
from prefect.testing.utilities import prefect_test_harness


@flow
def my_flow(name: str) -> str:
    return f"Hello, {name}!"


def test_my_flow():
    """Test that my_flow returns the expected greeting."""
    result = my_flow(name="World")
    assert result == "Hello, World!"
```

### 2. Testing a Task

```python
from prefect import task


@task
def add(a: int, b: int) -> int:
    return a + b


def test_add_task():
    """Tasks can be called directly in tests without a flow context."""
    result = add.fn(1, 2)  # Use .fn to call the underlying function
    assert result == 3
```

### 3. Using the Test Harness

For tests that require a full Prefect context (e.g., state tracking, persistence):

```python
import pytest
from prefect.testing.utilities import prefect_test_harness


@pytest.fixture(autouse=True, scope="session")
def prefect_test_harness_fixture():
    with prefect_test_harness():
        yield
```

### 4. Mocking External Services

```python
from unittest.mock import MagicMock, patch
import pytest
from prefect import flow, task


@task
def fetch_from_api(url: str) -> dict:
    import httpx
    response = httpx.get(url)
    return response.json()


@flow
def data_pipeline(url: str) -> dict:
    return fetch_from_api(url)


def test_data_pipeline_with_mock():
    mock_response = MagicMock()
    mock_response.json.return_value = {"key": "value"}

    with patch("httpx.get", return_value=mock_response):
        result = data_pipeline(url="https://api.example.com/data")

    assert result == {"key": "value"}
```

### 5. Testing Flow State

```python
from prefect import flow
from prefect.states import Completed, Failed


@flow
def failing_flow():
    raise ValueError("Something went wrong")


def test_flow_fails_gracefully():
    state = failing_flow(return_state=True)
    assert state.is_failed()
    assert "Something went wrong" in str(state.result(raise_on_failure=False))
```

### 6. Parametrized Tests

```python
import pytest
from prefect import flow


@flow
def multiply(a: int, b: int) -> int:
    return a * b


@pytest.mark.parametrize("a,b,expected", [
    (2, 3, 6),
    (0, 5, 0),
    (-1, 4, -4),
    (10, 10, 100),
])
def test_multiply(a, b, expected):
    assert multiply(a=a, b=b) == expected
```

## Fixtures

### Common Fixtures (conftest.py)

```python
import pytest
from prefect.testing.utilities import prefect_test_harness


@pytest.fixture(scope="session", autouse=True)
def prefect_db():
    """Provide an in-memory Prefect database for the test session."""
    with prefect_test_harness():
        yield


@pytest.fixture
def mock_api_response():
    """Provide a standard mock API response."""
    return {"status": "ok", "data": []}
```

## Best Practices

1. **Isolate tests**: Each test should be independent and not rely on shared mutable state.
2. **Use `.fn` for unit-testing tasks**: Call `my_task.fn(...)` to bypass Prefect's task execution machinery when you only want to test the logic.
3. **Prefer `prefect_test_harness` for integration tests**: This spins up an ephemeral in-memory database and API server.
4. **Mock external I/O**: Always mock HTTP calls, database connections, and filesystem operations in unit tests.
5. **Test failure paths**: Ensure flows and tasks behave correctly when dependencies fail or raise exceptions.
6. **Keep tests fast**: Unit tests should run in milliseconds; reserve slower integration tests for a separate suite.

## Running Tests

```bash
# Run all tests
pytest tests/

# Run with coverage
pytest tests/ --cov=prefect --cov-report=html

# Run only unit tests (exclude integration)
pytest tests/ -m "not integration"

# Run a specific test file
pytest tests/test_flows.py -v
```
