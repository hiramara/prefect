# Code Review Skill

This skill helps Claude perform thorough, constructive code reviews for the Prefect codebase.

## Overview

When reviewing code, Claude should evaluate:
- Correctness and logic
- Prefect-specific patterns and conventions
- Performance considerations
- Test coverage
- Documentation quality

## Review Process

### 1. Understand Context

Before reviewing, understand:
- What problem does this change solve?
- What is the scope of the change (bug fix, feature, refactor)?
- Are there related issues or PRs referenced?

### 2. Check Prefect Conventions

Prefect-specific patterns to verify:

```python
# Flows should use the @flow decorator
from prefect import flow, task

@flow(name="my-flow", description="Clear description of what this flow does")
def my_flow(param: str) -> str:
    result = my_task(param)
    return result

# Tasks should be atomic and reusable
@task(retries=3, retry_delay_seconds=10)
def my_task(param: str) -> str:
    return param.upper()
```

### 3. Review Checklist

#### Functionality
- [ ] Does the code do what it claims to do?
- [ ] Are edge cases handled?
- [ ] Is error handling appropriate?
- [ ] Are retries/timeouts configured where needed?

#### Code Quality
- [ ] Is the code readable and well-named?
- [ ] Is there unnecessary complexity?
- [ ] Are there any obvious performance issues?
- [ ] Is there duplicated code that could be refactored?

#### Prefect-Specific
- [ ] Are flows and tasks properly decorated?
- [ ] Is state handling correct?
- [ ] Are results cached appropriately?
- [ ] Are concurrency limits respected?
- [ ] Is logging used via `get_run_logger()`?

#### Testing
- [ ] Are there unit tests for new functionality?
- [ ] Do tests cover happy path and error cases?
- [ ] Are Prefect test utilities used correctly?

```python
# Example of proper Prefect testing
from prefect.testing.utilities import prefect_test_harness

def test_my_flow():
    with prefect_test_harness():
        result = my_flow("test")
        assert result == "TEST"
```

#### Documentation
- [ ] Are docstrings present for public functions?
- [ ] Are complex sections commented?
- [ ] Is the CHANGELOG updated if needed?

### 4. Providing Feedback

Structure feedback clearly:

**Blocking issues** (must fix before merge):
- Security vulnerabilities
- Correctness bugs
- Breaking changes without version bump

**Suggestions** (non-blocking improvements):
- Performance optimizations
- Code style improvements
- Additional test cases

**Nitpicks** (minor style preferences):
- Naming conventions
- Comment wording
- Minor formatting

### 5. Common Prefect Anti-Patterns

Flag these patterns as issues:

```python
# BAD: Calling tasks outside of flows
result = my_task()  # This won't be tracked by Prefect

# GOOD: Tasks called within flows
@flow
def my_flow():
    result = my_task()

# BAD: Using print() instead of Prefect logger
def my_task():
    print("Processing...")  # Not captured in Prefect logs

# GOOD: Use get_run_logger()
from prefect import get_run_logger

@task
def my_task():
    logger = get_run_logger()
    logger.info("Processing...")

# BAD: Mutable default arguments
@task
def process_items(items: list = []):  # Dangerous!
    pass

# GOOD: Use None as default
@task
def process_items(items: list = None):
    if items is None:
        items = []
```

## Output Format

When providing a review, use this structure:

```
## Code Review Summary

**Overall Assessment**: [Approve / Request Changes / Comment]

### Blocking Issues
1. [Issue description with file:line reference]
   - Why it's a problem
   - Suggested fix

### Suggestions
1. [Suggestion with file:line reference]
   - Rationale
   - Example if helpful

### Nitpicks
- [Minor items]

### Positive Notes
- [What was done well]
```
