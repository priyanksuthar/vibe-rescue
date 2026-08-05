# Vibe Rescue

Automated test generation, execution, and stabilization for any codebase.

Vibe Rescue analyzes your code changes, generates targeted tests across 10 categories (7 standard + 3 infrastructure), runs them against your local Docker setup, evaluates failures, and loops until stable -- catching the bugs that slip through to higher environments. With `--pro-mode`, it also validates real database schemas, ClickHouse tables, and Kafka topics against your code models to catch infrastructure drift.

## Installation

Copy the `vibe-rescue/` directory into your project's `.claude/skills/`:

```bash
cp -r path/to/vibe-rescue .claude/skills/vibe-rescue
```

The `templates/` subdirectory must travel with the skill files -- it contains reusable infrastructure test stubs.

## Quick Start

```bash
# After making code changes:
vibe-rescue

# Bootstrap tests for a service with no tests:
vibe-rescue --bootstrap my-service

# Auto-fix loop (no confirmation prompts):
vibe-rescue --auto-run

# Combine: bootstrap + auto-fix:
vibe-rescue --bootstrap my-service --auto-run

# Force re-scan project structure:
vibe-rescue --rescan

# Infrastructure validation (requires Docker containers):
vibe-rescue --pro-mode

# Bootstrap + infra validation:
vibe-rescue --bootstrap my-service --pro-mode

# Combine all: bootstrap + infra + auto-fix:
vibe-rescue --bootstrap my-service --pro-mode --auto-run

# Full E2E with dependent services:
vibe-rescue --pro-mode --all
```

## What It Does

### 6-Phase Pipeline

1. **Discover** -- Auto-detect project type, tech stack, services, communication graph
2. **Analyze** -- Classify git diff changes, resolve downstream dependencies
3. **Generate** -- Create/update tests in 7 category files per service
4. **Infrastructure** -- Start required Docker containers automatically
5. **Execute** -- Run tests in dependency order with early termination
6. **Report** -- Generate detailed report with coverage summary

### 7 Test Categories

| Category | What it catches |
|----------|----------------|
| `config_consistency` | Env vars referenced but not defined, config mismatches |
| `feature_flags` | Flags defined but unused, default value traps |
| `db_integrity` | Columns added but not all code paths updated |
| `api_contracts` | Missing field validation, wrong status codes |
| `message_contracts` | Payload changes breaking consumers, old events in queues |
| `feature_flows` | Business logic bugs, missing error handling |
| `cross_service` | Producer/consumer drift, new topics not consumed |

### Infrastructure Categories (--pro-mode)

| Category | What it catches |
|----------|----------------|
| `infra_postgres` | Tables/columns missing from DB, nullable mismatches, FK drift, orphaned columns |
| `infra_clickhouse` | Missing tables, column type mismatches, missing distributed tables/materialized views |
| `infra_kafka` | Topics not created, naming convention violations, broker unreachable, config misalignment |

These require real Docker containers running and are auto-skipped without `--pro-mode`.

### Two Execution Modes

- **Interactive (default):** Shows proposed fixes, waits for your approval
- **Auto-run (`--auto-run`):** Fixes automatically, stops after 3 failed attempts or on regression

## Configuration

Create `.vibe-rescue.yaml` at your project root (all fields optional):

```yaml
project:
  type: multi-service-monorepo
  services:
    - name: my-service
      path: ./my-service
      test_command: "pytest tests/ -v"

classification:
  skip:
    patterns: ["scripts/", "tools/", "benchmarks/"]

infrastructure:
  compose_file: ./docker-compose.yml
  dev_helper: ./dev.sh

categories:
  disabled: ["cross_service"]  # skip specific categories

auto_run:
  max_fix_attempts: 3
  revert_on_regression: true

reports:
  output_dir: "{service}/docs/test-reports"

pro_mode:
  container_orchestrator: "ucp.sh"  # or "docker-compose"
  postgres_env_prefix: "POSTGRES_"
  clickhouse_env_prefix: "CLICKHOUSE_"
  kafka_env_var: "KDC_KAFKA_BROKER_LIST"
```

## Templates

The `templates/` directory contains reusable test stubs for infrastructure validation:

| Template | Purpose | Customization Points |
|----------|---------|---------------------|
| `conftest_pro_mode.py` | pytest hooks + connection fixtures | `# ADAPT:` connection env vars, database names |
| `test_infra_postgres.py` | Schema drift detection | `# ADAPT:` model imports, model lists |
| `test_infra_clickhouse.py` | ClickHouse validation | `# ADAPT:` table names, expected columns |
| `test_infra_kafka.py` | Topic existence | `# ADAPT:` consumed/produced topic lists |
| `setup_cfg_markers.txt` | Marker registration | Copy into service's `setup.cfg` |

### Test Dependencies for --pro-mode

Add to `requirements.test.txt`:

```
psycopg2-binary>=2.9.0    # Sync PG driver for schema introspection
clickhouse-driver>=0.2.0  # Sync ClickHouse driver for schema queries
kafka-python>=2.0.2       # KafkaAdminClient for topic validation
```

## Supported Tech Stacks

| Language | Frameworks | Test Frameworks | Message Brokers | Databases |
|----------|-----------|-----------------|-----------------|-----------|
| Python | FastAPI, Flask, Django, Sanic | pytest, unittest | Kafka, RabbitMQ, SQS | PostgreSQL, MySQL, MongoDB, ClickHouse |
| Node.js | Express, Fastify, NestJS | Jest, Mocha, Vitest | Kafka, RabbitMQ, Bull | PostgreSQL, MySQL, MongoDB |
| Go | Gin, Echo, Chi | go test | Kafka, NATS, RabbitMQ | PostgreSQL, MySQL, MongoDB |
| Java | Spring Boot | JUnit, TestNG | Kafka, RabbitMQ | PostgreSQL, MySQL, MongoDB |

## Plugin Integration

Vibe Rescue works standalone or integrates with:

- **Superpowers:** Uses systematic-debugging for root cause analysis, dispatching-parallel-agents for multi-service parallelization, reads spec files for backward compatibility rules
- **Custom plugins:** Extension points for spec resolution, fix strategy, agent spawning, report publishing, CI integration

## Safety Rails

- Hand-written tests are NEVER auto-modified (always asks)
- Backward compatibility violations always stop execution
- Max 3 fix attempts per issue before escalating
- Fixes that cause regressions are automatically reverted
- Infrastructure drift is NEVER auto-fixed (always escalated with diagnostics)
- Scripts folder is always excluded from test generation

## License

MIT
