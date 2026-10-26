# Agentic Commerce Platform

A staged commerce project: build Java Spring Boot microservices one at a time, prove checkout works, then add a Python agent that investigates failed orders through service APIs.

## Current status

This is the **documentation baseline**. The repository contains design documents, a synthetic recovery scenario, and dependency files for the future Python environment. No microservices, notebooks, Python agents, or automated test suite are currently implemented.

Review and commit this baseline first. Implementation starts afterward, with one service per increment.

## Implementation order

1. **Inventory service** — stock queries, reservations, releases, persistence, and concurrency tests.
2. **Payment service** — simulated approval, decline, unknown outcomes, and idempotent payment attempts.
3. **Order service** — order lifecycle and checkout across inventory and payments.
4. **Integration** — local service orchestration and end-to-end failure/restart tests.
5. **Python agent** — read-only order investigation first; controlled recovery later.

Complete, test, review, and commit each increment before starting the next. Start with synthetic data, one warehouse, three SKUs, INR, and a fake payment provider.

## Documentation

| Document | Purpose |
|---|---|
| [Architecture](docs/ARCHITECTURE.md) | Service ownership and the eventual recovery-agent system |
| [Agent use cases](docs/AGENT_USE_CASES.md) | Later investigation and recovery capabilities |
| [Failure cases](docs/FAILURE_CASES.md) | Failure behavior and correctness expectations |
| [Recovery scenario](scenarios/complex_order_recovery.json) | Synthetic advanced scenario for later evaluation |

Follow the implementation order above. The architecture and advanced scenario describe future capabilities.

## Repository structure

```text
docs/                 Architecture, agent use cases, and failure cases
scenarios/            Synthetic scenario fixtures
pyproject.toml        Future Python environment dependencies
uv.lock               Locked Python dependencies
.env.example          Future model/tracing configuration template
```

`services/`, `agents/`, and infrastructure files will be added only when their implementation stages begin.

## Development setup

Reading and reviewing the baseline requires no running services, database, or model credentials. Validate the scenario with:

```bash
python3 -m json.tool scenarios/complex_order_recovery.json
```

The planned service toolchain is Java 21, Spring Boot, Maven, and PostgreSQL. The first service increment will pin the framework version and provide a Maven Wrapper and exact run/test commands. There is no application startup or build command yet.

For the future Python agent environment, Python 3.12 and uv are configured:

```bash
uv sync --locked
```

This installs dependencies only. Model access and hosted tracing remain opt-in. Keep credentials in an untracked `.env`; local payments remain simulated.
