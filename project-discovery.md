# Project Discovery

Auto-detect project type, tech stack, communication graph, file classification, test infrastructure, and feature flag patterns. Output a cached project profile at `.vibe-rescue/project-profile.yaml`.

## When to Run

- First invocation (no cached profile exists)
- User passes `--rescan` flag
- Structural changes detected: new/removed service directories, changed docker-compose, changed dependency files (requirements.txt, package.json, go.mod, pom.xml, Cargo.toml)

## Step 1: Detect Project Type

Scan the project root directory. Classify based on the FIRST matching rule:

| Check | Classification |
|-------|---------------|
| Workspace config found (`pnpm-workspace.yaml`, `lerna.json`, `Cargo.toml` with `[workspace]`, root `build.gradle` with `include`) | `monorepo-workspace` |
| Multiple subdirectories each containing their own dependency file (`requirements.txt`, `package.json`, `go.mod`, `pom.xml`) | `multi-service-monorepo` |
| Single dependency file at root only | `single-service` |

To find service directories, run:
```bash
find . -maxdepth 3 -name "requirements.txt" -o -name "package.json" -o -name "go.mod" -o -name "pom.xml" -o -name "Cargo.toml" | grep -v node_modules | grep -v vendor | grep -v .venv
```

Exclude directories named: `node_modules`, `vendor`, `.venv`, `venv`, `__pycache__`, `.git`, `scripts`, `docs`, `examples`.

## Step 2: Detect Tech Stack Per Service

For each service/module found in Step 1, detect:

### Language

| Indicator | Language |
|-----------|---------|
| `requirements.txt`, `setup.py`, `setup.cfg`, `pyproject.toml` | Python |
| `package.json` | Node.js / TypeScript |
| `go.mod` | Go |
| `pom.xml`, `build.gradle` | Java |
| `Cargo.toml` | Rust |

### Web Framework

Scan the service's dependency file:

**Python** (check `requirements.txt`, `setup.cfg install_requires`, `pyproject.toml`):
| Dependency | Framework |
|-----------|-----------|
| `fastapi` | FastAPI |
| `flask` | Flask |
| `django` | Django |
| `sanic` | Sanic |
| `tornado` | Tornado |
| `aiohttp` | aiohttp |

**Node.js** (check `package.json` dependencies):
| Dependency | Framework |
|-----------|-----------|
| `express` | Express |
| `fastify` | Fastify |
| `@nestjs/core` | NestJS |
| `koa` | Koa |
| `hapi`, `@hapi/hapi` | Hapi |

**Go** (scan import statements in `.go` files):
| Import | Framework |
|--------|-----------|
| `github.com/gin-gonic/gin` | Gin |
| `github.com/labstack/echo` | Echo |
| `github.com/go-chi/chi` | Chi |
| `net/http` only | stdlib |

**Java** (check `pom.xml` or `build.gradle`):
| Dependency | Framework |
|-----------|-----------|
| `spring-boot-starter-web` | Spring Boot |
| `quarkus` | Quarkus |
| `micronaut` | Micronaut |

### Test Framework

| Indicator | Test Framework |
|-----------|---------------|
| `pytest` in deps, `conftest.py` exists, `setup.cfg [tool:pytest]` | pytest |
| `unittest` imports in test files | unittest |
| `jest` in devDependencies, `jest.config.*` exists | Jest |
| `vitest` in devDependencies, `vitest.config.*` exists | Vitest |
| `mocha` in devDependencies | Mocha |
| `*_test.go` files present | go test |
| `junit` in deps, `@Test` annotations | JUnit |
| `testng` in deps | TestNG |

### Message Broker

Scan dependency files for broker client libraries:

| Dependency | Broker |
|-----------|--------|
| `confluent-kafka`, `kafka-python`, `aiokafka` | Kafka |
| `kafkajs` | Kafka |
| `github.com/segmentio/kafka-go`, `github.com/IBM/sarama` | Kafka |
| `pika`, `aio-pika` | RabbitMQ |
| `amqplib`, `amqp-connection-manager` | RabbitMQ |
| `boto3` (with SQS usage) | SQS |
| `@aws-sdk/client-sqs` | SQS |
| `nats`, `nats.py` | NATS |
| `bull`, `bullmq` | Redis Queue |

If a broker is detected, scan source files for topic/queue names:
```bash
# For Python Kafka
grep -rn "topic" {service_path}/  --include="*.py" | grep -i "kafka\|producer\|consumer\|topic" | grep -oP "'[a-z][-a-z0-9_.]+'" | sort -u
```

Adapt the grep pattern per language and broker.

### Database & ORM

| Dependency | Database | ORM |
|-----------|----------|-----|
| `sqlalchemy`, `asyncpg`, `psycopg2` | PostgreSQL | SQLAlchemy |
| `django.db` | PostgreSQL/MySQL/SQLite | Django ORM |
| `prisma`, `@prisma/client` | PostgreSQL/MySQL | Prisma |
| `typeorm` | PostgreSQL/MySQL | TypeORM |
| `sequelize` | PostgreSQL/MySQL | Sequelize |
| `gorm.io/gorm` | PostgreSQL/MySQL | GORM |
| `pymongo`, `motor` | MongoDB | pymongo/motor |
| `mongoose` | MongoDB | Mongoose |
| `clickhouse-driver`, `asynch` | ClickHouse | native |
| `alembic` | (migration tool) | (with SQLAlchemy) |
| `knex` | (migration tool) | (with any Node ORM) |
| `flyway`, `liquibase` | (migration tool) | (Java) |

### Config Pattern

Scan for config loading approaches:
| Pattern | Config Type |
|---------|------------|
| `os.getenv`, `os.environ` in Python | env_vars |
| `process.env` in Node.js | env_vars |
| `os.Getenv` in Go | env_vars |
| `argparse` in Python | argparse |
| `pydantic.BaseSettings` | pydantic_settings |
| `python-dotenv`, `dotenv` | dotenv |
| `viper` in Go | viper |
| YAML/JSON config file loading | config_file |

### Container Setup

| File Found | Container Tool |
|-----------|---------------|
| `docker-compose.yml` or `docker-compose.yaml` or `compose.yml` | docker-compose |
| `Dockerfile` | docker |
| `Makefile` with docker targets | make + docker |
| Shell scripts (`*.sh`) referencing docker | helper script |

Find the compose file:
```bash
find . -maxdepth 3 -name "docker-compose*.yml" -o -name "docker-compose*.yaml" -o -name "compose.yml" -o -name "compose.yaml" | head -5
```

Find dev helper scripts:
```bash
find . -maxdepth 3 -name "*.sh" -type f | xargs grep -l "docker\|compose" 2>/dev/null | head -5
```

If docker-compose found, extract profiles:
```bash
grep -oP 'profiles:\s*\[.*?\]' {compose_file} | sort -u
```

Extract infra service names (databases, brokers, caches):
```bash
grep -E "image:.*?(postgres|mysql|redis|mongo|kafka|zookeeper|rabbitmq|clickhouse|nats|memcached|elasticsearch)" {compose_file}
```

### Infrastructure Connections (for --pro-mode)

Scan for connection configuration env vars to detect which infrastructure services the service connects to:

```bash
# PostgreSQL connections
grep -rn "POSTGRES_\|DATABASE_URL\|SQLALCHEMY_DATABASE_URI" {service_path}/ --include="*.py" | grep -oP "(POSTGRES_\w+|DATABASE_URL|SQLALCHEMY_DATABASE_URI)" | sort -u

# ClickHouse connections
grep -rn "CLICKHOUSE_\w+_HOST\|CLICKHOUSE_\w+_PORT" {service_path}/ --include="*.py" | grep -oP "CLICKHOUSE_\w+" | sort -u

# Kafka broker connections
grep -rn "KAFKA_BROKER\|KAFKA_BOOTSTRAP\|KAFKA_SERVERS\|BROKER_LIST" {service_path}/ --include="*.py" --include="*.ts" --include="*.go" --include="*.java" | grep -oP "(\w*KAFKA\w*BROKER\w*|\w*KAFKA\w*BOOTSTRAP\w*|\w*KAFKA\w*SERVERS\w*)" | sort -u
```

Record each detected connection with its env var name. These are used by `--pro-mode` to determine which infrastructure containers to start and which test categories to enable.

## Step 3: Discover Communication Graph

Build edges between services:

### Message Broker Edges
For each service, cross-reference `topics_produced` and `topics_consumed`. If service A produces topic X and service B consumes topic X, create edge: `A -> B via {broker} on topic X`.

### HTTP Edges
Scan for HTTP client calls between services:
```bash
# Python
grep -rn "requests\.\|httpx\.\|aiohttp\.ClientSession\|AsyncClient" {service_path}/ --include="*.py" | grep -v test
# Node.js
grep -rn "fetch\|axios\|got\|node-fetch" {service_path}/ --include="*.ts" --include="*.js" | grep -v test | grep -v node_modules
```

### Shared Database Edges
If two services reference the same database name in their config, they share a database.

### Docker-Compose Edges
Parse `depends_on` from docker-compose to find declared dependencies.

### Documentation Edges
Search for structured documentation:
```bash
find . -maxdepth 4 -name "CLAUDE.md" -o -name "service.yaml" -o -name "architecture.md" -o -name "dependency-map.md" | head -10
```

If found, parse for service-to-service relationships. Documentation is highest priority for dependency data.

## Step 4: Build File Classification Map

For each service, build classification rules based on what was detected:

1. **API layer:** Find files that import the detected web framework's router/endpoint registration. Record their directory patterns.
2. **Message broker layer:** Find files that import the detected broker's consumer/producer classes. Record their directory patterns.
3. **Data layer:** Find files that import the detected ORM's model base class. Record their directory patterns.
4. **Config layer:** Find files that load configuration (env vars, config files). Record their patterns.
5. **Business logic:** Everything else that isn't tests, scripts, config, or infra.
6. **Scripts:** Always skip `*/scripts/`, `*/bin/`, `*/tools/`.

Run this analysis:
```bash
# Example for Python/FastAPI
grep -rln "from fastapi import\|from fastapi.routing" {service_path}/ --include="*.py" | xargs -I{} dirname {} | sort -u
# Example for Python/SQLAlchemy
grep -rln "from sqlalchemy\|from sqlalchemy.orm\|declarative_base\|DeclarativeBase" {service_path}/ --include="*.py" | xargs -I{} dirname {} | sort -u
```

## Step 5: Detect Existing Test Infrastructure

```bash
# Find test directories
find {service_path} -type d -name "tests" -o -name "test" -o -name "__tests__" -o -name "spec" | grep -v node_modules | grep -v vendor

# Find test config files
find {service_path} -maxdepth 2 -name "setup.cfg" -o -name "pytest.ini" -o -name "conftest.py" -o -name "jest.config.*" -o -name "vitest.config.*" -o -name ".mocharc.*"

# Find test run commands
grep -l "pytest\|jest\|vitest\|mocha\|go test" {service_path}/Makefile {service_path}/package.json {service_path}/*.sh 2>/dev/null

# Find existing test helpers/fixtures
find {service_path} -path "*/tests/conftest.py" -o -path "*/tests/helpers/*" -o -path "*/tests/fixtures/*" -o -path "*/tests/factories/*" -o -path "*/__tests__/setup.*"
```

## Step 6: Detect Feature Flag Pattern

```bash
# Python
grep -rn "os\.getenv\|os\.environ\.get" {service_path}/ --include="*.py" | grep -viE "database|host|port|password|secret|key|url|broker|redis" | head -20
# Node.js
grep -rn "process\.env\." {service_path}/ --include="*.ts" --include="*.js" | grep -viE "DATABASE|HOST|PORT|PASSWORD|SECRET|KEY|URL|BROKER|REDIS" | grep -v node_modules | head -20
# Feature flag libraries
grep -rn "launchdarkly\|unleash\|flagsmith\|feature.flag\|FeatureFlag" {service_path}/ --include="*.py" --include="*.ts" --include="*.js" --include="*.go" --include="*.java" | head -10
```

## Output Format

Save the discovered profile to `.vibe-rescue/project-profile.yaml`:

```yaml
# Auto-generated by vibe-rescue. Do not edit manually.
# Re-generate with: vibe-rescue --rescan
# Generated: {YYYY-MM-DD HH:MM:SS}

project_type: {single-service | multi-service-monorepo | monorepo-workspace}

services:
  - name: {service-dir-name}
    path: ./{relative-path}
    language: {python | nodejs | go | java | rust}
    framework: {fastapi | flask | django | sanic | express | nestjs | gin | spring-boot | ...}
    test_framework: {pytest | jest | vitest | go-test | junit | ...}
    test_dir: ./{path-to-test-dir}
    test_command: "{detected test command}"
    message_broker: {kafka | rabbitmq | sqs | nats | redis-queue | none}
    topics_produced: [{list of topic strings}]
    topics_consumed: [{list of topic strings}]
    databases: [{postgresql | mysql | mongodb | clickhouse | ...}]
    orm: {sqlalchemy | django-orm | prisma | typeorm | gorm | mongoose | none}
    config_pattern: {env_vars | argparse | pydantic_settings | dotenv | viper | config_file}
    feature_flag_pattern: {env_vars | launchdarkly | unleash | flagsmith | config_file | database | none}
    health_check: "{detected health endpoint or 'none'}"

communication_graph:
  - from: {service-name}
    to: {service-name}
    via: {kafka | rabbitmq | http | shared_db}
    topic: {topic-name}  # or endpoint for http

infrastructure:
  container_tool: {docker-compose | make | helper-script | none}
  compose_file: {path or "none"}
  dev_helper: {path or "none"}
  profiles: [{list of detected profiles}]
  infra_services: [{list of infra container names}]

  infrastructure_connections:
    postgres:
      - env_var: "{POSTGRES_CONNECTION_VAR}"
        database: "{extracted_db_name}"
    clickhouse:
      - env_var: "{CLICKHOUSE_HOST_VAR}"
        database: "{extracted_db_name}"
    kafka:
      - env_var: "{KAFKA_BROKER_VAR}"

file_classification:
  api_layer:
    patterns: [{list of detected patterns}]
    detected_from: "{what triggered detection}"
  message_broker_layer:
    patterns: [{list}]
    detected_from: "{reason}"
  data_layer:
    patterns: [{list}]
    detected_from: "{reason}"
  config_layer:
    patterns: [{list}]
    detected_from: "{reason}"
  business_logic:
    patterns: [{list}]
    detected_from: "remaining source files"
  scripts:
    patterns: ["*/scripts/", "*/bin/", "*/tools/"]
    action: SKIP
```

After writing the profile, create the directory if needed:
```bash
mkdir -p .vibe-rescue
```

## Step 7: Write Infrastructure to Memory (--pro-mode only)

If `--pro-mode` is active and this is a first run or `--rescan`, write discovered infrastructure connections to `.vibe-rescue/memory.yaml` in addition to the project profile.

Read `memory-schema.md` for the full memory file schema and auto-discovery agent steps.

After project discovery completes, extract the following from the detected profile and write to the `infrastructure` section of memory.yaml:

    infrastructure:
      postgres:
        - env_var: "{detected PostgreSQL env var}"
          database: "{extracted database name}"
          default_url: "{constructed connection URL}"
      clickhouse:
        - env_var: "{detected ClickHouse host env var}"
          database: "{detected database name}"
      kafka:
        - env_var: "{detected Kafka broker env var}"
          default_brokers: "localhost:9092"
      container_orchestrator: "{detected from infrastructure.dev_helper or infrastructure.container_tool}"
      compose_file: "{detected from infrastructure.compose_file}"
      test_environment_detected: "{true if compose_file or dev_helper found}"

Also detect and write naming patterns:

    patterns:
      topic_naming_regex: "{regex derived from common prefix/suffix of detected topics}"
      topic_prefix: "{common prefix if all topics share one, null otherwise}"
      consumer_config_path: "{path to consumer config file if found}"

Present the discovery summary box for user confirmation before writing. Only write after user confirms.
