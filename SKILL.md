---
name: vibe-rescue
description: Use after code changes or before commits to auto-generate targeted tests, run them, evaluate failures, and stabilize. Supports any tech stack (Python/Node/Go/Java). Use --bootstrap to generate baseline tests for a service from scratch. Use --auto-run for autonomous fix loop. Use --pro-mode to run infrastructure validation tests against real Docker services. Detects project structure, Kafka/RabbitMQ topics, DB schemas, API contracts, feature flags automatically.
---

# Vibe Rescue

Automated test generation, execution, and stabilization for any codebase.

**Announce at start:** "Using vibe-rescue to analyze changes and generate targeted tests."

## When to Use

**ONLY when code files are being changed (created, modified, deleted):**
- After applying code changes to source files, before committing
- When adding a new feature and want regression coverage
- When modifying message broker payloads, API contracts, or DB schemas
- To bootstrap test coverage for a service that has none
- As part of verification-before-completion flow after implementation

## When NOT to Use

**Do NOT invoke vibe-rescue during research or exploration:**
- Reading files to understand the codebase
- Exploring project structure, architecture, or patterns
- Investigating bugs or tracing data flows (use systematic-debugging instead)
- Reviewing code, reading docs, or answering questions
- Brainstorming, planning, or designing features
- Any activity that does NOT result in code file changes

**The trigger is file changes, not knowledge gathering.** If no source files have been written or modified in the current task, vibe-rescue has nothing to test.

## Entry Points

Parse the user's invocation for flags:

| Flag | Mode | Behavior |
|------|------|----------|
| (none) | Change mode, interactive | Analyze git diff, generate tests, run, show fixes for approval |
| `--bootstrap {service}` | Bootstrap mode | Scan full service, generate baseline tests across all categories |
| `--auto-run` | Autonomous | Auto-fix code/tests in a loop until stable (no per-fix confirmation) |
| `--rescan` | Rescan | Invalidate cached project profile AND memory file, re-run all discovery |
| `--pro-mode` | Infrastructure validation | Generate + run infra tests against real Docker services (PostgreSQL, ClickHouse, Kafka) |
| `--pro-mode --all` | Full E2E validation | Also start dependent service profiles for cross-service E2E tests |
| `--help` | Help | Print usage, flags, categories, and examples. No tests run. |

Combinable: `--bootstrap my-service --auto-run` = bootstrap + autonomous fix loop.
Combinable: `--bootstrap my-service --pro-mode` = bootstrap all categories + infra validation.

## Help Output

When the user passes `--help`, print the following block exactly and stop. Do NOT run any pipeline phases.

```
╔══════════════════════════════════════════════════════════════════╗
║  VIBE RESCUE                                                     ║
║  Automated test generation, execution, and stabilization         ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  USAGE                                                           ║
║    /vibe-rescue [flags] [target]                                 ║
║                                                                  ║
║  FLAGS                                                           ║
║    (none)              Analyze git diff, generate tests,         ║
║                        run interactively                         ║
║    --bootstrap <svc>   Scan full service, generate baseline      ║
║                        tests across all categories               ║
║    --auto-run          Auto-fix loop (no per-fix confirmation)   ║
║    --rescan            Re-detect project structure (invalidate   ║
║                        cached profile)                           ║
║    --pro-mode          Run infrastructure validation tests       ║
║                        against real Docker services              ║
║    --pro-mode --all    Also start dependent service profiles     ║
║                        for cross-service E2E tests               ║
║    --help              Show this help message                    ║
║                                                                  ║
║  COMBINABLE EXAMPLES                                             ║
║    /vibe-rescue --bootstrap my-svc --auto-run                    ║
║    /vibe-rescue --bootstrap my-svc --pro-mode                    ║
║    /vibe-rescue --bootstrap my-svc --pro-mode --auto-run         ║
║    /vibe-rescue --pro-mode --all                                 ║
║                                                                  ║
║  TEST CATEGORIES (standard)                                      ║
║    config_consistency   Env vars referenced but not defined      ║
║    feature_flags        Flags defined but unused, default traps  ║
║    db_integrity         Model/schema mismatches                  ║
║    api_contracts        Missing validation, wrong status codes   ║
║    message_contracts    Payload changes breaking consumers       ║
║    feature_flows        Business logic bugs                      ║
║    cross_service        Producer/consumer drift                  ║
║                                                                  ║
║  TEST CATEGORIES (--pro-mode, requires Docker)                   ║
║    infra_postgres       Table/column/FK drift vs real DB         ║
║    infra_clickhouse     Missing tables, type mismatches          ║
║    infra_kafka          Topic existence, naming conventions      ║
║                                                                  ║
║  PIPELINE PHASES                                                 ║
║    0.5 Memory     Load memory, permissions, auto-discovery       ║
║    0. Discover    Auto-detect project type and tech stack        ║
║    1. Analyze     Classify changes, resolve dependencies         ║
║    2. Generate    Create/update test files per category          ║
║    3. Infra       Start required Docker containers               ║
║    4. Execute     Run tests in dependency order                  ║
║    5. Fix         Evaluate failures, propose/apply fixes         ║
║    6. Report      Generate summary report                        ║
║                                                                  ║
║  CONFIGURATION                                                   ║
║    .vibe-rescue.yaml    Project-level overrides (optional)       ║
║                                                                  ║
║  TEMPLATES (for --pro-mode)                                      ║
║    templates/conftest_pro_mode.py      pytest hooks + fixtures   ║
║    templates/test_infra_postgres.py    Schema drift detection    ║
║    templates/test_infra_clickhouse.py  ClickHouse validation     ║
║    templates/test_infra_kafka.py       Topic validation          ║
║    templates/setup_cfg_markers.txt     Marker registration       ║
║                                                                  ║
║  SAFETY RAILS                                                    ║
║    - Hand-written tests are NEVER auto-modified                  ║
║    - Backward compat violations always stop execution            ║
║    - Max 3 fix attempts per issue before escalating              ║
║    - Regressions from fixes are auto-reverted                    ║
║    - All tests re-run after every fix                            ║
║    - Test functions are never deleted                            ║
║    - Infrastructure drift is never auto-fixed                    ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

## The 6-Phase Pipeline

Execute phases in order. Do not skip phases.

### Phase 0.5: Memory Check

Read `memory-schema.md` for full instructions.

1. Check for `.vibe-rescue/memory.yaml`
2. If exists:
   a. Load infrastructure config, preferences, patterns
   b. Show permission banner if `permissions` section has any `true` values:
      ```
      ╔══════════════════════════════════════════════════════════════════╗
      ║  VIBE RESCUE - Permission Review                                 ║
      ╠══════════════════════════════════════════════════════════════════╣
      ║  Previous session granted these permissions:                     ║
      ║                                                                  ║
      ║    [1] auto_run             Auto-fix code/tests without asking   ║
      ║    [2] auto_fix_test_infra  Auto-fix broken test imports/fixtures║
      ║                                                                  ║
      ║  Re-approve for this session? (yes / no / select)                ║
      ╚══════════════════════════════════════════════════════════════════╝
      ```
   c. Wait for permission re-approval ("yes" = all approved, "no" = all revoked, "select" = pick individually)
   d. Show active preferences summary: "Active preferences: {list}"
3. If not exists AND `--pro-mode`:
   a. Run auto-discovery agent (see `memory-schema.md` Auto-Discovery section)
   b. Present discovery summary for confirmation
   c. Create `.vibe-rescue/memory.yaml` with confirmed values
4. If not exists AND no `--pro-mode`:
   a. Continue to Phase 0 normally

### Phase 0: Project Discovery

Read `project-discovery.md` for full instructions.

1. Check for cached profile at `.vibe-rescue/project-profile.yaml`
2. If cache exists and no structural changes detected (docker-compose, dependency files, new/removed service dirs), use cached profile
3. If no cache or `--rescan` flag, run full discovery and save to `.vibe-rescue/project-profile.yaml`
4. Check for user overrides in `.vibe-rescue.yaml` at project root -- merge overrides into discovered profile

### Phase 1: Change Analysis

Read `change-analyzer.md` for full instructions.

**Change mode:**
1. Run `git diff --name-only` (staged + unstaged) to get changed files
2. Exclude files matching `scripts/` patterns from the project profile
3. Classify each file using the `file_classification` map from the project profile
4. For each classified change, determine which test categories to trigger
5. Use the `communication_graph` from the project profile to identify downstream service impacts
6. Look for the most recent spec file in `{service}/docs/specs/` to extract backward compatibility rules
7. Output: structured change manifest

**Bootstrap mode:**
1. Scan all source files in the target service directory
2. Map every API endpoint, message topic, DB model, config entry, feature flag
3. Enable ALL 7 test categories
4. Output: change manifest with all categories enabled

**Pro-mode addition (both modes):**
If `--pro-mode` flag is set, add 3 infrastructure test categories to the manifest regardless of what files changed:
- `infra_postgres` -- if service has SQLAlchemy models
- `infra_clickhouse` -- if service references ClickHouse tables
- `infra_kafka` -- if service has Kafka consumer/producer config

Infra categories are always full-set: drift happens from external migrations, not just code changes.

### Phase 2: Test Generation

Read `test-generators.md` for full instructions.

For each test category in the change manifest:
1. Check if the category file exists in the service's test directory
2. If not, create it with standard imports and fixtures matching the project's existing test patterns
3. Generate test functions following the rules in `test-generators.md` for that category
4. Append new test functions (never overwrite existing)
5. Use naming convention: `test_{category}_{what}_{scenario}`

**Multi-service:** If multiple services are affected, check if `dispatching-parallel-agents` skill is available (superpowers). If yes, spawn one agent per service. If no, run sequentially.

**Pro-mode test generation:**
If infra categories are in the manifest:
1. Check for `templates/` directory in the skill installation
2. Copy and adapt template files (`conftest_pro_mode.py`, `test_infra_postgres.py`, etc.) to the service's test directory
3. Replace `# ADAPT:` markers with service-specific values (model imports, topic lists, table names)
If `.vibe-rescue/memory.yaml` exists, use values from `infrastructure` and `patterns` sections to auto-populate `# ADAPT:` markers instead of leaving them for manual editing. If memory has no value for a marker, leave the `# ADAPT:` comment as fallback.
4. Add `--pro-mode` machinery to the service's `conftest.py` if not already present
5. Add infra markers to `setup.cfg` `[tool:pytest]` section
6. Add test dependencies to `requirements.test.txt`: `psycopg2-binary>=2.9.0`, `clickhouse-driver>=0.2.0`, `kafka-python>=2.0.2`

### Phase 3: Infrastructure Management

1. Determine which containers are needed based on test categories in the manifest
2. Check which containers are already running (`docker ps`)
3. Use the `infrastructure` section of the project profile to start missing containers
4. Wait for health checks before proceeding
5. Report readiness

Container requirements per category:
- `config_consistency`: None (static analysis)
- `feature_flags`: Same as the category the flag gates
- `api_contracts`: Service container only
- `db_integrity`: Database container(s)
- `message_contracts`: Message broker container(s)
- `feature_flows`: All infra for that service
- `cross_service`: Multiple services + all shared infra
- `infra_postgres`: PostgreSQL container
- `infra_clickhouse`: ClickHouse container
- `infra_kafka`: Kafka + Zookeeper containers

**Pro-mode container orchestration:**
Read `container_orchestrator` from `.vibe-rescue/memory.yaml`. If it is a script path, verify the script exists and use it for container lifecycle. If it is `docker-compose`, use docker-compose directly. If no orchestrator is configured, try docker-compose as fallback:
1. `docker compose up -d` for infrastructure containers
2. Wait for healthchecks: `pg_isready`, `clickhouse-client --query "SELECT 1"`, `kafka-broker-api-versions`
3. Set environment variables for host-side test runner (connection strings pointing to localhost)

### Phase 4: Test Execution

Run tests in this order (early termination on foundational failures):

1. `test_config_consistency` -- static, no containers
2. `test_feature_flags` -- flag validation
3. `test_db_integrity` -- schema correctness (STOP if fails)
4. `test_api_contracts` -- API contracts
5. `test_message_contracts` -- broker contracts
6. Existing hand-written tests (regression)
7. `test_feature_flows` -- feature flows
8. `test_cross_service` -- cross-service flows (last)
9. `test_infra_postgres` -- schema drift validation (--pro-mode only)
10. `test_infra_clickhouse` -- ClickHouse table/column validation (--pro-mode only)
11. `test_infra_kafka` -- topic existence + naming convention (--pro-mode only)

Use the `test_command` from the project profile to run tests.

If `config_consistency` or `db_integrity` fail, STOP execution and go directly to Phase 5.
If `test_infra_postgres` fails with missing critical tables, STOP and go to Phase 5.
Without `--pro-mode`, steps 9-11 are auto-skipped (zero impact on standard runs).

### Phase 5: Evaluate & Fix

Read `fix-loop.md` for full instructions.

**Default mode (no --auto-run):**
1. For each failure, classify as TEST issue, CODE issue, BACKWARD COMPAT issue, or HAND-WRITTEN test
2. Show proposed fix as a diff preview -- DO NOT APPLY
3. Display: "Tip: Run with --auto-run to automatically fix and re-run until stable"
4. Wait for user approval before applying each fix
5. After applying approved fix, re-run ALL tests (not just the fixed one)
6. Repeat until all pass or user stops

**--auto-run mode:**
1. For each failure, classify and auto-fix
2. Re-run ALL tests after each fix
3. If same issue fails 3 times: STOP, escalate to user with full context
4. If a fix introduces NEW failures: REVERT the fix, escalate to user
5. NEVER auto-modify hand-written tests -- always ask user even in auto-run
6. Loop until all tests pass

### Post-Phase 5: Preference Learning

After the fix loop completes, check if the user overrode any default behavior during this run:

1. If user skipped a test category: add to `preferences.skip_categories` in memory
2. If user changed max fix attempts: update `preferences.auto_run_max_attempts`
3. If user granted `--auto-run` mid-session: set `permissions.auto_run: true`
4. If user said "always use pro-mode": set `preferences.pro_mode_default: true`

After updating, print: "Saved: {preference description}. Clear with `--rescan`."

Update rules:
- Overwrite, not append (changing a preference replaces the old value)
- If user un-skips a previously skipped category, remove it from the list
- Memory file must stay under ~100 lines

### Phase 6: Report

Read `report-template.md` for full instructions.

1. Generate report at `{service}/docs/test-reports/YYYY-MM-DD-test-report.md`
2. Display summary in terminal
3. Ask user: keep containers running or stop?

## Extension Point Resolution

At the start of each run, detect which plugins are available:

```
Check: is superpowers installed?
  YES -> use spec_resolver: read_spec_file
         use fix_strategy: systematic_debugging_skill
         use agent_spawner: dispatching_parallel_agents
  NO  -> use spec_resolver: ask_user
         use fix_strategy: built_in_diff
         use agent_spawner: sequential
```

If `.vibe-rescue.yaml` exists at project root, its values override both defaults and auto-detected settings.

## Rationalization Blockers

These thoughts mean STOP -- you're skipping phases:

| Thought | Reality |
|---------|---------|
| "The change is too small for tests" | Small changes cause big breaks. Run the pipeline. |
| "I can see the tests will pass" | You haven't run them. Run Phase 4. |
| "The project profile is probably fine" | Check the cache. Phase 0 is fast when cached. |
| "I'll just fix the code without showing the diff" | Default mode requires user approval. Show the diff. |
| "This service has no downstream dependencies" | Check the communication graph. Don't guess. |
| "The old tests don't matter" | Check backward compatibility in the spec. |
| "I'll skip the report, user saw the output" | Always generate the report file. Phase 6 is not optional. |
| "The infra tests don't need to run, the models haven't changed" | Schema drift happens from migrations, not just model changes. Run with --pro-mode. |
