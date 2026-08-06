# Report Template

Generate a summary report after test execution and fix loop complete.

## Report Location

Save to: `{service_path}/docs/test-reports/YYYY-MM-DD-test-report.md`

Create the directory if it doesn't exist:
```bash
mkdir -p {service_path}/docs/test-reports
```

If `.vibe-rescue.yaml` specifies a `reports.output_dir`, use that instead.

## Report Template

Fill in all `{placeholders}` from the run data. Omit sections that are empty (e.g., no downstream services checked, no code fixes applied).

```markdown
# Vibe Rescue Report - {service_name}

**Date:** {YYYY-MM-DD} | **Mode:** {Change | Bootstrap} | **Execution:** {Interactive | Auto-run} | **Trigger:** {git diff summary | full scan}

## Project Profile

- **Type:** {project_type}
- **Language:** {language}
- **Framework:** {framework}
- **Test Framework:** {test_framework}
- **Message Broker:** {broker or "none"}
- **Database:** {database or "none"}

## Change Summary

| File | Change Type | Impact |
|------|-------------|--------|
| {file_path} | {api_change / message_change / db_change / config_change / logic_change} | {which test categories triggered} |

## Test Coverage

| Category | Tests | Passed | Failed | Auto-Fixed | Skipped |
|----------|-------|--------|--------|------------|---------|
| config_consistency | {n} | {n} | {n} | {n} | {n} |
| feature_flags | {n} | {n} | {n} | {n} | {n} |
| db_integrity | {n} | {n} | {n} | {n} | {n} |
| api_contracts | {n} | {n} | {n} | {n} | {n} |
| message_contracts | {n} | {n} | {n} | {n} | {n} |
| feature_flows | {n} | {n} | {n} | {n} | {n} |
| cross_service | {n} | {n} | {n} | {n} | {n} |
| **Existing (regression)** | **{n}** | **{n}** | **{n}** | **{n}** | **{n}** |
| **TOTAL** | **{n}** | **{n}** | **{n}** | **{n}** | **{n}** |

## Infrastructure Validation Tests (--pro-mode)

Run with `pytest --pro-mode` or via your project's container orchestrator (configured in `.vibe-rescue/memory.yaml`).
Auto-skipped without the flag -- zero impact on existing test runs.

| File | Tests | Description |
|------|-------|-------------|
| `tests/test_infra_postgres.py` | {n} | {description} |
| `tests/test_infra_clickhouse.py` | {n} | {description} |
| `tests/test_infra_kafka.py` | {n} | {description} |
| **Total infra tests** | **{n}** | Requires Docker containers running |

### Usage

    # Unit tests only (default, no Docker needed)
    pytest tests/ -o "addopts="

    # Infrastructure validation (requires Docker containers)
    pytest tests/ --pro-mode -o "addopts="

    # Via container orchestrator (if configured in .vibe-rescue/memory.yaml)
    {container_orchestrator} test {service_name}

    # With dependent services (E2E)
    {container_orchestrator} test {service_name} --all

## Test Fixes Applied

| # | Test | Category | Fix Description |
|---|------|----------|----------------|
| 1 | {test_function_name} | {category} | {what was changed and why} |

## Code Fixes Applied

| # | File:Line | Fix Description |
|---|-----------|----------------|
| 1 | {file_path}:{line} | {what was changed and why} |

## Downstream Services Checked

| Service | Reason | Status | Tests Run |
|---------|--------|--------|-----------|
| {service_name} | {consumes topic X} | {PASS / FAIL} | {n} tests |

## Backward Compatibility

- **Spec:** {spec_ref or "none found"}
- **Required:** {Yes / No / Unknown}
- **Status:** {VERIFIED / VIOLATION / NOT_CHECKED}
- **Details:** {what was checked and result}

## Existing Tests (Regression)

| Service | Total | Passed | Failed |
|---------|-------|--------|--------|
| {service_name} | {n} | {n} | {n} |

## Categories Not Applicable

{list categories skipped and why, e.g.:}
- **message_contracts** -- no message broker detected in project profile
- **cross_service** -- single-service project, no communication graph

## Fix Loop Summary

- **Total iterations:** {n}
- **Tests auto-fixed:** {n}
- **Code fixes applied:** {n}
- **Escalated to user:** {n}
- **Reverted fixes:** {n}
```

## Terminal Summary

After saving the report file, display a condensed version in the terminal:

```
╔══════════════════════════════════════════════════════╗
║  VIBE RESCUE REPORT - {service_name}                ║
╠══════════════════════════════════════════════════════╣
║  Mode: {mode}  |  Execution: {interactive/auto-run}  ║
║                                                      ║
║  Tests:  {total} total  |  {passed} passed  |  {failed} failed  ║
║  Fixed:  {test_fixes} test fixes  |  {code_fixes} code fixes    ║
║                                                      ║
║  Categories:                                         ║
║    config_consistency:  {status}                     ║
║    feature_flags:       {status}                     ║
║    db_integrity:        {status}                     ║
║    api_contracts:       {status}                     ║
║    message_contracts:   {status}                     ║
║    feature_flows:       {status}                     ║
║    cross_service:       {status}                     ║
║    regression:          {status}                     ║
║    infra_postgres:      {status}                     ║
║    infra_clickhouse:    {status}                     ║
║    infra_kafka:         {status}                     ║
║                                                      ║
║  Backward Compat: {VERIFIED/VIOLATION/NOT_CHECKED}   ║
║                                                      ║
║  Report: {report_file_path}                          ║
╚══════════════════════════════════════════════════════╝
```

Where `{status}` = `PASS (n/n)` or `FAIL (n/n)` or `SKIP` or `N/A`.

Infra categories only appear in the terminal summary when `--pro-mode` was used. Without it, they are omitted entirely (not shown as SKIP).

## Post-Report Actions

After displaying the report, ask:

1. "Keep Docker containers running? (yes/no)" -- if containers were started
2. If any tests were escalated: "There are {n} unresolved failures. Would you like to investigate them now?"
