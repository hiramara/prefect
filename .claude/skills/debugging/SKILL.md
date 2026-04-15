# Debugging Skill

This skill provides guidance on debugging Prefect flows, tasks, and deployments effectively.

## Overview

Debugging Prefect workflows involves understanding the execution model, inspecting state transitions, reading logs, and using the Prefect UI or CLI to diagnose issues.

## Common Debugging Patterns

### 1. Enable Verbose Logging

Always start by increasing log verbosity to capture more context:

```python
import logging
from prefect import flow, task, get_run_logger

@task
def my_task(value: int) -> int:
    logger = get_run_logger()
    logger.debug(f"Processing value: {value}")
    result = value * 2
    logger.info(f"Result: {result}")
    return result

@flow(log_prints=True)
def my_flow(values: list[int]) -> list[int]:
    logger = get_run_logger()
    logger.info(f"Starting flow with {len(values)} values")
    results = my_task.map(values)
    return results
```

### 2. Inspect Flow Run State

Use the Prefect client to inspect flow run states programmatically:

```python
from prefect.client.orchestration import get_client
import asyncio

async def inspect_flow_run(flow_run_id: str):
    async with get_client() as client:
        flow_run = await client.read_flow_run(flow_run_id)
        print(f"State: {flow_run.state.type}")
        print(f"Message: {flow_run.state.message}")
        
        task_runs = await client.read_task_runs(
            flow_run_filter=FlowRunFilter(id={"any_": [flow_run_id]})
        )
        for task_run in task_runs:
            print(f"Task {task_run.name}: {task_run.state.type}")

asyncio.run(inspect_flow_run("your-flow-run-id"))
```

### 3. Catch and Inspect Exceptions

Use `return_state=True` to capture task states without raising exceptions immediately:

```python
from prefect import flow, task
from prefect.states import Failed

@task
def risky_task(x: int) -> int:
    if x < 0:
        raise ValueError(f"Negative value not allowed: {x}")
    return x ** 2

@flow
def debug_flow(values: list[int]):
    logger = get_run_logger()
    
    for val in values:
        state = risky_task(val, return_state=True)
        if state.is_failed():
            logger.error(f"Task failed for value {val}: {state.message}")
            # Optionally inspect the exception
            try:
                state.result()
            except Exception as e:
                logger.error(f"Exception type: {type(e).__name__}, message: {e}")
        else:
            logger.info(f"Task succeeded for value {val}: {state.result()}")
```

### 4. Use Retries with Logging

Add retries with exponential backoff and log each retry attempt:

```python
from prefect import task
from prefect.tasks import exponential_backoff
import httpx

@task(
    retries=3,
    retry_delay_seconds=exponential_backoff(backoff_factor=2),
    retry_jitter_factor=0.5,
)
def fetch_data(url: str) -> dict:
    logger = get_run_logger()
    logger.info(f"Fetching data from {url} (attempt {task_run_context.task_run.run_count})")
    
    response = httpx.get(url, timeout=30)
    response.raise_for_status()
    return response.json()
```

### 5. Local Debugging Without Prefect Server

Run flows locally with `serve=False` to debug without needing a Prefect server:

```python
from prefect import flow, task
from prefect.testing.utilities import prefect_test_harness

@task
def add(a: int, b: int) -> int:
    return a + b

@flow
def addition_flow(a: int, b: int) -> int:
    return add(a, b)

# Run locally for debugging
if __name__ == "__main__":
    # Use test harness for isolated local execution
    with prefect_test_harness():
        result = addition_flow(1, 2)
        print(f"Result: {result}")
```

## Debugging Checklist

### Flow Fails on Startup
- [ ] Check that all required environment variables are set
- [ ] Verify infrastructure (work pool, worker) is running
- [ ] Confirm deployment configuration is valid with `prefect deployment inspect <name>`
- [ ] Check for import errors in flow module

### Task Fails Unexpectedly
- [ ] Review task logs in Prefect UI or via `prefect flow-run logs <id>`
- [ ] Check if the task is hitting resource limits (memory, CPU)
- [ ] Verify external service connectivity (databases, APIs)
- [ ] Inspect task run state message for error details

### Flow Hangs or Times Out
- [ ] Check for deadlocks in concurrent task execution
- [ ] Verify `task_runner` concurrency limits are appropriate
- [ ] Look for blocking I/O without timeouts
- [ ] Use `timeout_seconds` on tasks to enforce limits

### State Not Persisting
- [ ] Confirm result storage is configured correctly
- [ ] Check `persist_result=True` is set on tasks that need it
- [ ] Verify storage block credentials are valid

## CLI Debugging Commands

```bash
# View recent flow runs
prefect flow-run ls

# Get logs for a specific flow run
prefect flow-run logs <flow-run-id>

# Inspect a deployment
prefect deployment inspect "my-flow/my-deployment"

# Check work pool status
prefect work-pool inspect <pool-name>

# View worker status
prefect worker ls
```

## Environment Variables for Debugging

```bash
# Enable debug logging
export PREFECT_LOGGING_LEVEL=DEBUG

# Log all SQL queries (useful for diagnosing server issues)
export PREFECT_LOGGING_SERVER_LEVEL=DEBUG

# Disable result caching during debugging
export PREFECT_TASKS_REFRESH_CACHE=true

# Set API URL for local server
export PREFECT_API_URL=http://127.0.0.1:4200/api
```

## Tips

1. **Use `log_prints=True`** on flows to automatically capture `print()` statements in Prefect logs.
2. **Tag flow runs** with debug metadata: `flow.with_options(tags=["debug", "local"])()`.
3. **Pause flows** mid-execution with `pause_flow_run()` to inspect intermediate state.
4. **Use artifacts** to store intermediate results for inspection: `create_markdown_artifact()`.
5. **Profile slow flows** by checking task run durations in the Prefect UI timeline view.
