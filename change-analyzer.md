# Change Analyzer

Classify changed files and resolve cross-service dependencies to produce a change manifest.

## Change Mode: Git Diff Analysis

### Step 1: Get Changed Files

```bash
# Staged + unstaged changes
git diff --name-only HEAD
# Include untracked new files
git ls-files --others --exclude-standard
```

Combine both lists. Remove duplicates.

### Step 2: Filter Out Excluded Paths

Remove files matching skip patterns from the project profile's `file_classification.scripts.patterns`. Always skip:
- `*/scripts/*`
- `*/bin/*`
- `*/tools/*`
- `*.md` (documentation)
- `*.txt` (non-code)
- `*/tests/*` (existing test files -- we generate separately)

Also check `.vibe-rescue.yaml` for additional skip patterns under `classification.skip.patterns`.

### Step 3: Group Files by Service

For each remaining changed file, determine which service it belongs to by matching against the `services[].path` entries in the project profile. Files not under any service path are flagged as "root-level" changes.

### Step 4: Classify Each File

For each changed file, match against the `file_classification` map from the project profile. Walk the patterns in this priority order (first match wins):

1. **api_layer** → change_type: `api_change` → triggers: `api_contracts`
2. **message_broker_layer** → change_type: `message_change` → triggers: `message_contracts`, `cross_service`
3. **data_layer** → change_type: `db_change` → triggers: `db_integrity`
4. **config_layer** → change_type: `config_change` → triggers: `config_consistency`, `feature_flags`
5. **business_logic** → change_type: `logic_change` → triggers: `feature_flows`
6. **scripts** → action: SKIP

If a file doesn't match any pattern, classify as `logic_change` (default to `feature_flows`).

### Step 5: Content-Level Analysis

Beyond path-based classification, scan the diff content for deeper signals:

```bash
git diff HEAD -- {file}
```

| Diff Content Pattern | Additional Category Triggered |
|---------------------|------------------------------|
| New env var reference (`os.getenv`, `process.env`, `os.Getenv`) not in config files | `feature_flags` |
| New topic string or queue name | `message_contracts`, `cross_service` |
| New column in ORM model or migration | `db_integrity` |
| Changed function signature that's imported by other services | `cross_service` |
| New HTTP endpoint/route | `api_contracts` |

### Pro-Mode Category Injection

When `--pro-mode` is passed, add all applicable infra categories to every affected service's `test_categories` list, regardless of what files changed:

| Infrastructure Connection Detected | Category Added |
|-------------------------------------|----------------|
| PostgreSQL (SQLAlchemy models found) | `infra_postgres` |
| ClickHouse (clickhouse imports or env vars found) | `infra_clickhouse` |
| Kafka (consumer/producer config found) | `infra_kafka` |

These are always full-set runs. Drift happens from external migrations, not just code changes, so there is no content-level analysis for infra categories.

### Step 6: Resolve Downstream Dependencies

Using the `communication_graph` from the project profile:

1. For each affected service, find all edges where this service is the `from` node
2. Each `to` service in those edges is a downstream impact
3. Record the reason (topic name, endpoint, shared DB)
4. Add `message_contracts` and `cross_service` to the downstream service's test categories

If no communication graph exists (standalone mode, single-service), skip this step.

### Step 7: Resolve Backward Compatibility

Look for the most recent spec file:

```bash
ls -t {service_path}/docs/specs/*.md 2>/dev/null | head -1
```

If found, scan the file for the "Backward Compatibility" section:

```bash
grep -A 10 "## Backward Compatibility" {spec_file}
```

Extract:
- `required`: Yes or No
- `affected_flows`: list of flows
- `migration_strategy`: how old data is handled

If no spec file exists:
- If superpowers is available: check `docs/superpowers/specs/` as fallback
- If standalone: will ask user during Phase 5 evaluation

### Output: Change Manifest

```yaml
mode: change  # or "bootstrap"
timestamp: "{YYYY-MM-DD HH:MM:SS}"

affected_services:
  - name: {service-name}
    path: {service-path}
    change_types: [{api_change, message_change, db_change, config_change, logic_change}]
    changed_files:
      - path: {file-path}
        classification: {api_layer | message_broker_layer | data_layer | config_layer | business_logic}
        change_type: {api_change | message_change | db_change | config_change | logic_change}
    test_categories: [{api_contracts, message_contracts, db_integrity, feature_flows, feature_flags, cross_service, config_consistency}]

downstream_impacts:
  - service: {downstream-service-name}
    reason: "{why this service is impacted}"
    via: {kafka | rabbitmq | http | shared_db}
    topic_or_endpoint: "{specific topic or endpoint}"
    test_categories: [{message_contracts, cross_service}]

backward_compatibility:
  required: {true | false | unknown}
  spec_ref: "{path to spec file or 'none'}"
  affected_flows: [{list}]
  migration_strategy: "{strategy or 'none'}"

pro_mode:
  enabled: {true | false}
  all_services: {true | false}
  infra_categories: [{infra_postgres, infra_clickhouse, infra_kafka}]
```

## Bootstrap Mode: Full Service Scan

In bootstrap mode, skip the git diff and instead:

1. List all source files in the target service (excluding tests, scripts, docs)
2. Classify every file using the same file_classification map
3. Enable ALL 7 test categories regardless of what's found
4. If `--pro-mode` is also passed, add the 3 infra categories (`infra_postgres`, `infra_clickhouse`, `infra_kafka`) based on detected infrastructure connections
5. Set `backward_compatibility.required: false` (no prior behavior to break)
5. Set `downstream_impacts: []` (bootstrap is per-service)

```bash
find {service_path} -name "*.py" -o -name "*.ts" -o -name "*.js" -o -name "*.go" -o -name "*.java" | grep -v tests | grep -v test | grep -v __tests__ | grep -v node_modules | grep -v vendor | grep -v scripts | grep -v __pycache__
```
