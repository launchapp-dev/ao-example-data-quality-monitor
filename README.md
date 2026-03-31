# data-quality-monitor

Automated data quality monitoring pipeline — profiles CSV/JSON files, detects anomalies against historical baselines, applies safe auto-fixes, and generates quality scorecards with trend tracking.

## Workflow Diagram

```
data/incoming/
      │
      ▼
┌─────────────────┐
│  discover-files │  (command: python3) — scan for CSV/JSON files
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  profile-data   │  (command: python3) — compute stats per file
└────────┬────────┘
         │
         ▼
┌──────────────────────┐
│  normalize-profiles  │  (agent: profiler/Haiku) — validate vs schemas
└────────┬─────────────┘
         │
         ▼
┌──────────────────────┐
│  detect-anomalies    │  (agent: anomaly-detector/Sonnet) — compare vs memory baselines
└────────┬─────────────┘
         │
         ▼
┌──────────────────────┐
│  review-anomalies    │  (agent: anomaly-detector/Sonnet)
└────────┬─────────────┘
    approve │ rework──────────────────────┐
            │                             │ (max 2x)
            ▼                             ▲
┌──────────────────────┐                  │
│    decide-fixes      │ ─── rework ──────┘
└────────┬─────────────┘
         │
         ▼
┌──────────────────────┐
│    apply-fixes       │  (agent: fixer/Haiku) — auto-fix only
└────────┬─────────────┘
         │
         ▼
┌──────────────────────┐
│  generate-scorecards │  (agent: reporter/Sonnet) — scorecards + dashboard
└──────────────────────┘
         │
         ▼
output/scorecards/*.md + output/dashboard.md
```

## Quick Start

```bash
cd examples/data-quality-monitor
ao daemon start

# Run full quality check on all files in data/incoming/
ao workflow run quality-check

# Run spot check on a single file
ao queue enqueue --title "customers.csv" --workflow-ref spot-check

# Watch live
ao daemon stream --pretty

# Check outputs
cat output/dashboard.md
cat output/scorecards/customers.csv.md
```

## Agents

| Agent | Model | Role |
|---|---|---|
| **profiler** | claude-haiku-4-5 | Reads Python profiling output, normalizes into structured findings JSON, checks against declared schemas |
| **anomaly-detector** | claude-sonnet-4-6 | Compares current stats vs memory baselines, classifies anomaly severity, decides fix strategy, self-reviews output |
| **fixer** | claude-haiku-4-5 | Applies only pre-approved auto-fix transformations (trim, coerce, normalize dates), logs every change |
| **reporter** | claude-sonnet-4-6 | Generates per-file scorecards and pipeline dashboard, computes quality scores, retrieves and stores trends |

## AO Features Demonstrated

1. **Command phases** — Python stdlib scripts for data profiling (no external deps)
2. **Multi-agent pipeline** — 4 specialized agents with distinct responsibilities
3. **Multi-model routing** — Haiku for fast/cheap extraction, Sonnet for reasoning and reporting
4. **Decision contracts** — `healthy|degraded|critical` quality verdict, `auto-fix|manual-review|escalate` fix decisions
5. **Rework loops** — Anomaly review can loop back to detection (max 2 attempts)
6. **Memory MCP** — Persistent quality baselines and run history across executions
7. **Scheduled automation** — Hourly quality checks via cron
8. **Auto-fix audit trail** — Every transformation logged with old/new values

## Requirements

No external API keys required — uses only:
- Python 3 (stdlib: csv, json, re, statistics, pathlib, hashlib)
- MCP servers (installed via npx automatically):
  - `@modelcontextprotocol/server-filesystem`
  - `@modelcontextprotocol/server-memory`
  - `@modelcontextprotocol/server-sequential-thinking`

## Directory Structure

```
data-quality-monitor/
├── .ao/workflows/          # AO workflow definitions
├── config/
│   ├── quality-config.yaml     # Thresholds, scoring weights, fix rules
│   └── schemas/                # Expected schemas per dataset
│       ├── customers.yaml
│       ├── transactions.yaml
│       └── inventory.yaml
├── data/
│   ├── incoming/           # Raw data files to monitor (CSV, JSON)
│   ├── pipeline/           # Intermediate artifacts (profiles, findings, anomalies)
│   └── fixed/              # Auto-fixed output files
├── output/
│   ├── scorecards/         # Per-file quality scorecards (markdown)
│   └── dashboard.md        # Pipeline-wide quality dashboard
├── CLAUDE.md
└── README.md
```

## Adding Your Own Data

Drop any `.csv` or `.json` file into `data/incoming/` and run `ao workflow run quality-check`.

To add a schema definition, create `config/schemas/{name}.yaml` with a `file_pattern` glob and `columns` list. The profiler agent will automatically match files against schemas.

## Sample Output

After a quality run, `output/dashboard.md` shows:

```
# Data Quality Dashboard — 2026-03-31T10:00:00Z

| File               | Score  | Status   | Anomalies | Auto-Fixes | Needs Review |
|--------------------|--------|----------|-----------|------------|--------------|
| customers.csv      | 74/100 | Degraded | 2         | 3          | 1            |
| transactions.json  | 61/100 | Degraded | 3         | 1          | 2            |

Average Score: 67.5 | Healthy: 0 | Degraded: 2 | Critical: 0

Top Issues:
1. [HIGH] transactions.json — null tx_id (1 row)
2. [MEDIUM] customers.csv — null spike in tier column (+10.2pp from baseline)
3. [LOW] customers.csv — date format violations in signup_date (3 rows)
```
