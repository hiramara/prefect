# Deployment Skill

This skill covers deploying Prefect flows to various infrastructure targets, managing work pools, and configuring deployment schedules.

## Overview

Prefect deployments allow you to:
- Schedule flows to run automatically
- Trigger flows via API or UI
- Configure infrastructure (Docker, Kubernetes, serverless)
- Manage concurrency and work pools

## Core Concepts

### Work Pools
Work pools route flow runs to specific infrastructure. Types include:
- `process` — runs locally as a subprocess
- `docker` — runs in Docker containers
- `kubernetes` — runs as Kubernetes jobs
- `ecs` / `cloud-run` / `azure-container-instance` — cloud-managed containers

### Workers
Workers poll work pools and execute flow runs. Start a worker with:
```bash
prefect worker start --pool my-work-pool
```

### Deployments
A deployment packages a flow with its configuration (schedule, parameters, infrastructure).

## Creating Deployments

### Using `flow.deploy()` (recommended)

```python
from prefect import flow

@flow(log_prints=True)
def my_workflow(name: str = "world"):
    print(f"Hello, {name}!")

if __name__ == "__main__":
    my_workflow.deploy(
        name="my-deployment",
        work_pool_name="my-process-pool",
        cron="0 9 * * *",  # daily at 9am
        parameters={"name": "Prefect"},
    )
```

### Using `prefect.yaml`

For more complex setups, define deployments in `prefect.yaml`:

```yaml
# prefect.yaml
name: my-project
prefect-version: 3.x

build: null
push: null

pull:
  - prefect.deployments.steps.git_clone:
      repository: https://github.com/my-org/my-repo.git
      branch: main

deployments:
  - name: daily-etl
    entrypoint: flows/etl.py:run_etl
    work_pool:
      name: my-docker-pool
    schedule:
      cron: "0 6 * * *"
      timezone: America/New_York
    parameters:
      env: production
```

Deploy with:
```bash
prefect deploy --name daily-etl
```

## Docker Deployments

```python
from prefect import flow
from prefect.docker import DockerImage

@flow(log_prints=True)
def containerized_flow(env: str = "dev"):
    print(f"Running in {env}")

if __name__ == "__main__":
    containerized_flow.deploy(
        name="containerized-deployment",
        work_pool_name="my-docker-pool",
        image=DockerImage(
            name="my-org/my-flow:latest",
            dockerfile="Dockerfile",
        ),
        push=True,  # push image to registry
    )
```

## Schedules

```python
from prefect.client.schemas.schedules import CronSchedule, IntervalSchedule
from datetime import timedelta

# Cron schedule
my_flow.deploy(
    name="cron-deployment",
    work_pool_name="my-pool",
    cron="30 8 * * MON-FRI",  # weekdays at 8:30am
    timezone="UTC",
)

# Interval schedule
my_flow.deploy(
    name="interval-deployment",
    work_pool_name="my-pool",
    interval=timedelta(hours=6),
    anchor_date="2024-01-01T00:00:00",
)
```

## Triggering Runs

### Via CLI
```bash
prefect deployment run 'my-workflow/my-deployment' --param name=Claude
```

### Via Python SDK
```python
from prefect.deployments import run_deployment

async def trigger_run():
    flow_run = await run_deployment(
        name="my-workflow/my-deployment",
        parameters={"name": "Claude"},
        timeout=300,  # wait up to 5 min for completion
    )
    print(f"Flow run state: {flow_run.state_name}")
```

## Best Practices

1. **Pin dependencies** — use `requirements.txt` or lock files in your deployment image
2. **Use tags** — tag deployments for filtering in the UI (`tags=["production", "etl"]`)
3. **Set concurrency limits** — prevent duplicate runs with work pool concurrency settings
4. **Parameterize environments** — pass `env` as a parameter rather than hardcoding
5. **Version deployments** — include git SHA or version in deployment name for traceability
6. **Use pull steps** — for remote workers, use `git_clone` or `s3_download` pull steps

## Common Patterns

### Multi-environment deployments
```python
import sys

ENV = sys.argv[1] if len(sys.argv) > 1 else "dev"

my_flow.deploy(
    name=f"etl-{ENV}",
    work_pool_name=f"{ENV}-pool",
    parameters={"env": ENV},
    tags=[ENV],
)
```

### Pausing / resuming schedules
```bash
prefect deployment pause 'my-workflow/my-deployment'
prefect deployment resume 'my-workflow/my-deployment'
```

## Checklist

- [ ] Work pool exists and has an active worker
- [ ] Flow entrypoint is correct (`file.py:function_name`)
- [ ] Required infrastructure credentials are configured
- [ ] Schedule timezone is explicitly set
- [ ] Default parameters are sensible for automated runs
- [ ] Deployment tags reflect environment and team ownership
