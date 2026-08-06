# Memory Schema

Per-project memory file at `.vibe-rescue/memory.yaml`. Stores user-confirmed infrastructure connections, execution preferences, session permissions, and naming patterns. Complements `project-profile.yaml` (which stores auto-detected project structure).

## When to Read

- Phase 0.5 (Memory Check): load before any pipeline phase
- Phase 2 (Test Generation): read `infrastructure` and `patterns` to populate template `# ADAPT:` markers
- Phase 3 (Infrastructure Management): read `container_orchestrator` and `compose_file`
- Phase 5 (Fix Loop): check `permissions.auto_run` and `permissions.auto_fix_test_infra`

## When to Write

- After auto-discovery agent confirms infrastructure (Section: Auto-Discovery below)
- After user overrides a default preference during a run (Section: Self-Learning below)
- After user grants a new permission mid-session (Section: Permissions below)

## File Schema

```yaml
# Auto-managed by vibe-rescue. Manual edits are respected.
# Last updated: {YYYY-MM-DD}

infrastructure:
  postgres:
    - env_var: DATABASE_URL
      database: my_app_db
      default_url: "postgresql://user:password@localhost:5432/my_app_db"
  clickhouse:
    - env_var: CLICKHOUSE_HOST
      database: analytics
  kafka:
    - env_var: KAFKA_BOOTSTRAP_SERVERS
      default_brokers: "localhost:9092"
  mongodb:
    - env_var: MONGO_URI
  redis:
    - env_var: REDIS_URL
  container_orchestrator: docker-compose
  compose_file: ./docker-compose.yml
  test_environment_detected: true

preferences:
  skip_categories: []
  test_order_override: null
  auto_run_max_attempts: 3
  report_format: markdown
  pro_mode_default: false

permissions:
  auto_run: false
  auto_fix_test_infra: false

patterns:
  topic_naming_regex: "^[a-z0-9-]+$"
  topic_prefix: null
  consumer_config_path: null
```

## Size Cap

The memory file must never exceed ~100 lines. The skill prunes deprecated or redundant entries on each write. No history, no logs -- only the current confirmed state.

## Update Semantics

- **Overwrite, not append.** Changing a preference replaces the old value.
- **Confirm once.** After writing a new preference, print: "Saved: {preference}. Clear with `--rescan`."
- **`--rescan` clears all.** Resets memory.yaml completely, triggering fresh discovery.

## Auto-Discovery Agent

Runs when `--pro-mode` is invoked and memory has no `infrastructure` section, or `--rescan` is passed, or `test_environment_detected` is `false`.

### Step 1: Confirm Project Path

Single question: "I'll scan `{detected_project_root}` for infrastructure config. Correct?"

### Step 2: Package Scan

Parse dependency files to detect infrastructure packages:

| Language | Dependency Files | Packages Detected |
|----------|------------------|--------------------|
| Python | `requirements.txt`, `setup.cfg`, `pyproject.toml` | `psycopg2`, `asyncpg`, `sqlalchemy`, `clickhouse-driver`, `asynch`, `confluent-kafka`, `aiokafka`, `kafka-python`, `pymongo`, `motor`, `redis`, `pika`, `aio-pika` |
| Node.js | `package.json` | `pg`, `knex`, `typeorm`, `prisma`, `sequelize`, `kafkajs`, `amqplib`, `ioredis`, `bullmq`, `mongoose` |
| Go | `go.mod` | `github.com/lib/pq`, `gorm.io/gorm`, `github.com/segmentio/kafka-go`, `github.com/IBM/sarama`, `github.com/go-redis/redis`, `go.mongodb.org/mongo-driver` |
| Java | `pom.xml`, `build.gradle` | `spring-boot-starter-data-jpa`, `postgresql`, `spring-kafka`, `spring-boot-starter-data-mongodb`, `spring-boot-starter-data-redis`, `clickhouse-jdbc` |

### Step 3: Connection Env Var Scan

Grep source files for env var access patterns per language:

| Language | Patterns to scan |
|----------|-----------------|
| Python | `os.getenv("X")`, `os.environ.get("X")`, `os.environ["X"]` |
| Node.js | `process.env.X`, `process.env["X"]` |
| Go | `os.Getenv("X")`, `viper.GetString("X")` |
| Java | `@Value("${X}")`, `System.getenv("X")`, `Environment.getProperty("X")` |

Match detected env vars against infrastructure patterns:
- `*DATABASE*`, `*POSTGRES*`, `*PG_*` -> PostgreSQL
- `*CLICKHOUSE*`, `*CH_*` -> ClickHouse
- `*KAFKA*`, `*BROKER*` -> Kafka
- `*REDIS*`, `*CACHE*` -> Redis
- `*MONGO*` -> MongoDB

### Step 4: Test Environment Detection

Before asking the user how to run containers, check if the project already has test infrastructure:

```bash
# Docker compose with infra services
find . -maxdepth 3 -name "docker-compose*.yml" -o -name "compose*.yml" | head -5

# Test-specific compose
find . -maxdepth 3 -name "docker-compose.test.yml" -o -name "docker-compose.ci.yml" | head -5

# Dev helper scripts
find . -maxdepth 3 -name "*.sh" -type f | xargs grep -l "docker\|compose\|pytest\|test" 2>/dev/null | head -5

# Makefile with test targets
grep -l "test\|docker" Makefile */Makefile 2>/dev/null | head -5

# Existing conftest.py with connection fixtures
grep -rn "create_engine\|KafkaAdminClient\|clickhouse_driver\|Client(" */tests/conftest.py **/conftest.py 2>/dev/null | head -10
```

If existing test infrastructure is found, use those settings directly. Flag `test_environment_detected: true`.

### Step 5: Topic and Table Name Detection

**Kafka topics:**
```bash
grep -rnoP "\"[a-z][-a-z0-9_.]+\"" {service_path}/ --include="*.py" --include="*.ts" --include="*.go" --include="*.java" | grep -i "topic\|consumer\|producer\|subscribe" | head -30
```

Auto-detect naming convention regex. If all topics share a prefix, store in `patterns.topic_prefix`.

**DB tables:**
```bash
grep -rn "__tablename__\|@Entity\|@Table\|tableName" {service_path}/ --include="*.py" --include="*.ts" --include="*.go" --include="*.java" | grep -v test | head -20
```

**ClickHouse tables:**
```bash
grep -rn "FROM\|INSERT INTO\|CREATE TABLE" {service_path}/ --include="*.py" --include="*.sql" | grep -vi "test\|__pycache__" | head -20
```

### Step 6: Present Summary for Confirmation

Display the discovery summary box (see spec Section 2, Step 6). On "yes" write to memory. On "edit" ask which field to change. On "rescan" re-run.

### Step 7: Write to Memory

Store all confirmed values. Subsequent runs read from memory -- no re-discovery unless `--rescan`.

## Self-Learning Execution Preferences

### Learning Triggers

| User action during a run | Memory field updated |
|---|---|
| User skips a test category when prompted | `preferences.skip_categories` -- add category |
| User un-skips a previously skipped category | `preferences.skip_categories` -- remove category |
| User specifies test execution order | `preferences.test_order_override` -- set list |
| User changes max fix attempts | `preferences.auto_run_max_attempts` -- set value |
| User says "always use pro-mode" | `preferences.pro_mode_default` -- set `true` |
| User says "don't show terminal report" | `preferences.report_format` -- set `file_only` |
| User says "report to terminal only" | `preferences.report_format` -- set `terminal_only` |

### Storage Rules

1. **Overwrite, not append.** Changing a preference replaces the old value.
2. **Confirm once.** Print: "Saved: {description}. Clear with `--rescan`."
3. **`--rescan` clears preferences.** Fresh start.
4. **No history.** Only the current value.

### Application Rules

1. At run start, after loading memory, list active preferences.
2. Preferences are suggestions, not locks. User can override with explicit flags.
3. Explicit flags always win.

## Session-Scoped Permissions

### Permission Banner

When memory has permissions from a previous session, show the banner box (see spec Section 4). Ask user to re-approve for this session.

### Permission Definitions

| Permission | Scope | Controls |
|---|---|---|
| `auto_run` | Session-only | Auto-fix code/tests without confirmation |
| `auto_fix_test_infra` | Session-only | Auto-fix broken test imports/fixtures |

### Behavior

- **"yes"**: All permissions re-approved for this session.
- **"no"**: All revoked. Interactive mode.
- **"select"**: User picks individually.
- **No previous permissions**: Banner not shown.
- **Mid-session grant**: Stored in memory for next session's banner.
- **Never auto-escalate**: Resets when session ends.
