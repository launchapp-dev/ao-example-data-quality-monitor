# Data Quality Monitor — Build Plan

## Overview

Data quality monitoring pipeline — profiles CSV/JSON data files for schema consistency,
detects anomalies (null spikes, distribution shifts, format violations), validates against
business rules, auto-fixes safe issues (trimming, type coercion), generates quality
scorecards, and tracks quality trends over time via memory MCP.

All data profiling uses `python3 -c` and `jq` command phases — no external APIs needed.
Agent phases handle anomaly analysis, fix decisions, and scorecard generation.

---

## Agents (4)

| Agent | Model | Role |
|---|---|---|
| **profiler** | claude-haiku-4-5 | Fast extraction — reads profiling output, normalizes stats into structured JSON |
| **anomaly-detector** | claude-sonnet-4-6 | Analyzes profiles against baselines, classifies anomalies by severity |
| **fixer** | claude-haiku-4-5 | Applies safe auto-fixes (trim whitespace, coerce types, fill defaults) and logs changes |
| **reporter** | claude-sonnet-4-6 | Generates quality scorecards, trend reports, and summary dashboards |

### MCP Servers Used by Agents

- **filesystem** — all agents read/write data files and reports
- **memory** (@modelcontextprotocol/server-memory) — anomaly-detector and reporter store/retrieve quality baselines and historical trends
- **sequential-thinking** — anomaly-detector uses for multi-factor anomaly reasoning

---

## Workflows (2)

### 1. `quality-check` (primary — triggered per data quality run)

Main pipeline: data directory in -> scorecards + fixes + trend report out.

**Phases:**

1. **discover-files** (command)
   - Command: `python3 scripts/discover-files.py`
   - Scans `data/incoming/` for CSV and JSON files
   - Writes file manifest to `data/pipeline/file-manifest.json`
   - Each entry: `{ path, format, size_bytes, modified_at, row_count_estimate }`

2. **profile-data** (command)
   - Command: `python3 scripts/profile-data.py`
   - For each file in manifest, computes:
     - Column names, types, null counts, null percentages
     - Unique value counts, min/max/mean for numerics
     - String length stats, format pattern detection (email, date, phone, etc.)
     - Row count, duplicate row count
     - Schema fingerprint (ordered column name + type hash)
   - Writes per-file profile to `data/pipeline/profiles/{filename}.json`
   - Writes aggregate summary to `data/pipeline/profiles/summary.json`

3. **normalize-profiles** (agent: profiler)
   - Reads all profile JSONs from `data/pipeline/profiles/`
   - Normalizes into consistent structure with:
     - `schema_check`: column presence/type vs expected schema from `config/schemas/`
     - `null_report`: columns with nulls, percentages, whether above threshold
     - `value_ranges`: columns outside expected min/max
     - `format_violations`: strings not matching declared patterns
     - `duplicates`: duplicate row count and examples
   - Writes normalized findings to `data/pipeline/findings/{filename}.json`
   - Writes `data/pipeline/findings/summary.json`

4. **detect-anomalies** (agent: anomaly-detector)
   - Reads findings from `data/pipeline/findings/`
   - Retrieves historical baselines from memory MCP
   - Compares current run against baselines:
     - Null spike: null % increased by >10 points from baseline
     - Distribution shift: mean shifted >2 standard deviations
     - Schema drift: columns added/removed/type-changed
     - Volume anomaly: row count changed by >50% from baseline
   - Decision contract: `verdict` (healthy | degraded | critical), `reasoning`, `anomalies[]`
   - Writes anomaly report to `data/pipeline/anomalies/{filename}.json`
   - Stores updated baselines in memory MCP

5. **review-anomalies** (agent: anomaly-detector)
   - Decision contract: `verdict` (approve | rework), `reasoning`
   - Sanity-checks the anomaly detection run:
     - Are all files accounted for?
     - Do severity levels make sense relative to findings?
     - Any contradictions between anomalies and raw profiles?
   - **Routing:**
     - `approve` -> decide-fixes
     - `rework` -> detect-anomalies (max 2 attempts)

6. **decide-fixes** (agent: anomaly-detector)
   - Decision contract: `verdict` (auto-fix | manual-review | escalate), `reasoning`, `fix_plan[]`
   - For each anomaly, classifies fix action:
     - **auto-fix**: trim whitespace, coerce obvious types (string "123" -> int), fill known defaults, normalize date formats
     - **manual-review**: ambiguous type coercions, large null blocks, possible data corruption
     - **escalate**: schema drift (new/removed columns), >20% data loss risk, security-sensitive fields
   - Writes fix plan to `data/pipeline/fix-plan.json`

7. **apply-fixes** (agent: fixer)
   - Reads fix plan from `data/pipeline/fix-plan.json`
   - Only applies fixes marked `auto-fix`
   - For each fix:
     - Reads source file from `data/incoming/`
     - Applies transformation
     - Writes fixed file to `data/fixed/{filename}`
     - Logs every change: `{ file, row, column, old_value, new_value, fix_type }`
   - Writes fix log to `data/pipeline/fix-log.json`
   - Capabilities: writes_files, mutates_state

8. **generate-scorecards** (agent: reporter)
   - Reads anomalies, findings, fix logs, and baselines
   - Generates per-file quality scorecard: `output/scorecards/{filename}.md`
     - Overall quality score (0-100)
     - Dimension breakdown: completeness, consistency, accuracy, timeliness
     - Anomalies found with severity
     - Fixes applied
     - Trend vs previous run (from memory)
   - Generates pipeline-wide dashboard: `output/dashboard.md`
     - Summary table: all files, scores, status
     - Aggregate metrics: avg quality, % healthy/degraded/critical
     - Top issues across all files
     - Trend chart (text-based) from memory baselines
   - Stores run scores in memory MCP for trend tracking
   - Capabilities: writes_files, mutates_state, requires_commit

### 2. `spot-check` (on-demand — single file)

Quick single-file quality check.

**Phases:**

1. **profile-single** (command)
   - Command: `python3 scripts/profile-data.py --file {{subject_title}}`
   - Profiles a single file from `data/incoming/`
   - Writes to `data/pipeline/profiles/{filename}.json`

2. **analyze-single** (agent: anomaly-detector)
   - Reads profile, checks against baselines from memory
   - Generates inline scorecard
   - Writes scorecard to `output/scorecards/{filename}.md`

---

## Data Model

### Config Files (static — read-only reference)

| File | Content |
|---|---|
| `config/quality-config.yaml` | Global thresholds, scoring weights, auto-fix rules |
| `config/schemas/customers.yaml` | Expected schema for customers dataset |
| `config/schemas/transactions.yaml` | Expected schema for transactions dataset |
| `config/schemas/inventory.yaml` | Expected schema for inventory dataset |

### Data Files (mutable — written by agents/scripts)

| File | Content | Writers |
|---|---|---|
| `data/incoming/*.csv` | Raw data files to monitor | External (sample data) |
| `data/incoming/*.json` | Raw data files to monitor | External (sample data) |
| `data/pipeline/file-manifest.json` | Discovered files | discover-files |
| `data/pipeline/profiles/{file}.json` | Statistical profile per file | profile-data.py |
| `data/pipeline/profiles/summary.json` | Aggregate profile stats | profile-data.py |
| `data/pipeline/findings/{file}.json` | Normalized findings per file | profiler |
| `data/pipeline/findings/summary.json` | Aggregate findings | profiler |
| `data/pipeline/anomalies/{file}.json` | Anomaly report per file | anomaly-detector |
| `data/pipeline/fix-plan.json` | Fix decisions for all files | anomaly-detector |
| `data/pipeline/fix-log.json` | Applied fix audit log | fixer |
| `data/fixed/{file}` | Fixed data files | fixer |

### Output Files (generated artifacts)

| File | Content |
|---|---|
| `output/scorecards/{file}.md` | Per-file quality scorecard |
| `output/dashboard.md` | Pipeline-wide quality dashboard |

---

## Command Phase Scripts (2)

### `scripts/discover-files.py`
- Scans `data/incoming/` for `.csv` and `.json` files
- Computes size, modification time, estimates row count
- Outputs `data/pipeline/file-manifest.json`
- Pure Python (csv, json, os, pathlib — stdlib only)

### `scripts/profile-data.py`
- Reads file manifest (or single file via `--file` flag)
- For CSV files: column names, types, null counts, numeric stats, string length stats
- For JSON files: key presence, type distribution, nested depth, array lengths
- Detects format patterns (email regex, date patterns, phone patterns)
- Outputs per-file JSON profiles
- Pure Python (csv, json, re, statistics — stdlib only)
- Handles malformed rows gracefully (logs to stderr, continues)

---

## Schedules

| Schedule | Cron | Workflow |
|---|---|---|
| `hourly-quality-check` | `0 * * * *` (every hour) | quality-check |

(`spot-check` is triggered on-demand via queue)

---

## AO Features Demonstrated

1. **Command phases with real CLIs** — Python scripts for data profiling, jq for JSON validation
2. **Multi-agent pipeline** — 4 specialized agents: profiler, anomaly-detector, fixer, reporter
3. **Multi-model routing** — Haiku for fast extraction/fixes, Sonnet for analysis/reporting
4. **Decision contracts** — Quality verdict (healthy/degraded/critical), fix action (auto-fix/manual-review/escalate), review gate (approve/rework)
5. **Rework loops** — Anomaly review can send back to detection (max 2 attempts)
6. **Scheduled automation** — Hourly data quality checks via cron
7. **Memory MCP** — Persistent quality baselines and historical trends across runs
8. **Auto-fix pipeline** — Safe fixes applied automatically, risky ones flagged for human review
9. **Audit logging** — Every auto-fix is logged with old/new values for traceability

---

## Sample Data

### Sample Quality Config (`config/quality-config.yaml`)
```yaml
scoring:
  dimensions:
    completeness:
      weight: 30
      description: "Measures null rates and missing required fields"
    consistency:
      weight: 25
      description: "Measures schema adherence, type consistency, format compliance"
    accuracy:
      weight: 25
      description: "Measures value ranges, distribution normality, outlier rates"
    timeliness:
      weight: 20
      description: "Measures data freshness, update frequency, staleness"

thresholds:
  healthy: 80
  degraded: 50
  critical: 0

null_spike_threshold_pct: 10     # Alert if null % increases by this much
distribution_shift_stddev: 2.0   # Alert if mean shifts by this many stddevs
volume_change_threshold_pct: 50  # Alert if row count changes by this much

auto_fix_rules:
  trim_whitespace: true
  coerce_numeric_strings: true
  normalize_date_formats: true
  fill_defaults:
    enabled: true
    max_null_pct: 5              # Only fill defaults if <5% of column is null
  case_normalization: false       # Too risky by default
```

### Sample Schema Definition (`config/schemas/customers.yaml`)
```yaml
name: customers
file_pattern: "customers*.csv"
columns:
  - name: customer_id
    type: integer
    required: true
    unique: true
  - name: email
    type: string
    required: true
    format: email
  - name: name
    type: string
    required: true
    min_length: 1
    max_length: 200
  - name: signup_date
    type: date
    required: true
    format: "YYYY-MM-DD"
    min: "2020-01-01"
  - name: tier
    type: string
    required: false
    allowed_values: ["free", "basic", "premium", "enterprise"]
  - name: lifetime_value
    type: float
    required: false
    min: 0
    max: 1000000
```

### Sample Scorecard Output (`output/scorecards/customers.csv.md`)
```markdown
# Data Quality Scorecard: customers.csv

**Overall Score: 74/100** - Degraded

| Dimension | Score | Status |
|---|---|---|
| Completeness | 25/30 | Healthy |
| Consistency | 18/25 | Degraded |
| Accuracy | 19/25 | Healthy |
| Timeliness | 12/20 | Degraded |

## Anomalies Detected
1. **Null spike in `tier` column** (severity: medium)
   - Current: 12.3% null | Baseline: 2.1% null | Delta: +10.2pp
2. **Date format violations in `signup_date`** (severity: low)
   - 3 rows with format "MM/DD/YYYY" instead of "YYYY-MM-DD"

## Fixes Applied
- Trimmed whitespace in `name` column (47 rows affected)
- Normalized 3 date formats in `signup_date` to YYYY-MM-DD
- Coerced `lifetime_value` string "N/A" -> null (2 rows)

## Fixes Requiring Review
- `tier` column null spike — manual review needed (may indicate upstream data issue)

## Trend
- Previous score: 81/100 (2026-03-30)
- Change: -7 points
- Completeness trending down for 3 consecutive runs

---
*Run: 2026-03-31T10:00:00Z | [Dashboard](../dashboard.md)*
```

### Sample Incoming Data (`data/incoming/customers.csv`)
```csv
customer_id,email,name,signup_date,tier,lifetime_value
1001,alice@example.com,Alice Johnson,2024-03-15,premium,4500.00
1002,bob@test.co, Bob Smith ,2024-05-22,basic,1200.50
1003,charlie@demo.org,Charlie Brown,03/28/2024,free,0
1004,diana@example.com,Diana Prince,2024-07-01,,8900.00
1005,eve@nowhere.net,Eve Torres,2024-08-10,enterprise,N/A
1006,frank.invalid,Frank Castle,2024-09-01,premium,3200.00
1007,grace@example.com,,2024-10-15,basic,750.25
1008,hank@test.co,Hank Pym,2024-11-20,unknown,2100.00
```

### Sample Incoming Data (`data/incoming/transactions.json`)
```json
[
  {"tx_id": "TX-001", "customer_id": 1001, "amount": 299.99, "currency": "USD", "timestamp": "2026-03-30T14:22:00Z", "status": "completed"},
  {"tx_id": "TX-002", "customer_id": 1002, "amount": -50.00, "currency": "USD", "timestamp": "2026-03-30T15:01:00Z", "status": "refund"},
  {"tx_id": "TX-003", "customer_id": 9999, "amount": 150.00, "currency": "EUR", "timestamp": "2026-03-30", "status": "pending"},
  {"tx_id": null, "customer_id": 1003, "amount": 75.50, "currency": "USD", "timestamp": "2026-03-31T08:00:00Z", "status": "completed"},
  {"tx_id": "TX-005", "customer_id": 1004, "amount": "1200", "currency": "usd", "timestamp": "2026-03-31T09:15:00Z", "status": "COMPLETED"}
]
```

---

## Directory Structure

```
examples/data-quality-monitor/
├── .ao/workflows/
│   ├── agents.yaml
│   ├── phases.yaml
│   ├── workflows.yaml
│   ├── mcp-servers.yaml
│   └── schedules.yaml
├── config/
│   ├── quality-config.yaml
│   └── schemas/
│       ├── customers.yaml
│       ├── transactions.yaml
│       └── inventory.yaml
├── scripts/
│   ├── discover-files.py
│   └── profile-data.py
├── data/
│   ├── incoming/            # Raw data files to monitor
│   │   ├── customers.csv
│   │   └── transactions.json
│   ├── pipeline/
│   │   ├── profiles/
│   │   ├── findings/
│   │   └── anomalies/
│   └── fixed/               # Auto-fixed output files
├── output/
│   ├── scorecards/
│   └── dashboard.md
├── CLAUDE.md
└── README.md
```
