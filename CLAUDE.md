# Data Quality Monitor — Agent Context

## What This Project Does

Automated data quality monitoring pipeline. Given CSV and JSON data files in `data/incoming/`, it:
1. Profiles each file statistically (null rates, type distributions, format patterns, value ranges)
2. Normalizes findings against declared schemas in `config/schemas/`
3. Detects anomalies by comparing to historical baselines stored in memory MCP
4. Applies safe auto-fixes (whitespace, type coercion, date normalization)
5. Generates quality scorecards and a pipeline dashboard

## Key Paths

| Path | Purpose |
|---|---|
| `data/incoming/` | Raw source files — DO NOT modify directly |
| `data/pipeline/` | Intermediate pipeline artifacts (profiles, findings, anomalies, fix-plan, fix-log) |
| `data/fixed/` | Auto-fixed output files |
| `output/scorecards/` | Per-file quality scorecards |
| `output/dashboard.md` | Pipeline-wide summary |
| `config/quality-config.yaml` | Global thresholds and scoring weights |
| `config/schemas/` | Expected schema definitions (one per dataset type) |

## Pipeline Artifact Format

### Profile JSON (data/pipeline/profiles/{file}.json)
```json
{
  "filename": "customers.csv",
  "format": "csv",
  "row_count": 8,
  "column_count": 6,
  "columns": {
    "email": {
      "null_count": 0,
      "null_pct": 0.0,
      "unique_count": 8,
      "sample_values": ["alice@example.com", "bob@test.co"],
      "detected_formats": {"email": 0.875},
      "string": {"min_length": 10, "max_length": 24, "mean_length": 17.3}
    }
  },
  "duplicate_row_count": 0,
  "schema_fingerprint": "a1b2c3d4e5f6"
}
```

### Findings JSON (data/pipeline/findings/{file}.json)
```json
{
  "filename": "customers.csv",
  "schema_check": {
    "columns_expected": ["customer_id", "email", "name", "signup_date", "tier", "lifetime_value"],
    "columns_found": ["customer_id", "email", "name", "signup_date", "tier", "lifetime_value"],
    "columns_missing": [],
    "columns_extra": [],
    "type_mismatches": [{"column": "lifetime_value", "expected": "float", "found": "string", "example": "N/A"}]
  },
  "null_report": {
    "columns_with_nulls": [{"column": "tier", "null_count": 1, "null_pct": 12.5, "above_threshold": false}]
  },
  "value_ranges": {"out_of_range": []},
  "format_violations": [{"column": "email", "expected_format": "email", "violations_found": 1, "examples": ["frank.invalid"]}],
  "duplicates": {"duplicate_row_count": 0, "total_rows": 8, "duplicate_pct": 0.0},
  "schema_fingerprint": "a1b2c3d4e5f6"
}
```

### Anomaly JSON (data/pipeline/anomalies/{file}.json)
```json
{
  "filename": "customers.csv",
  "severity": "degraded",
  "first_run": false,
  "anomalies": [
    {
      "type": "null_spike",
      "severity": "medium",
      "column": "tier",
      "description": "Null rate increased by 10.2 percentage points from baseline",
      "current_value": 12.3,
      "baseline_value": 2.1,
      "delta": 10.2
    }
  ]
}
```

### Fix Plan (data/pipeline/fix-plan.json)
```json
[
  {
    "file": "customers.csv",
    "column": "name",
    "fix_action": "auto-fix",
    "fix_type": "trim_whitespace",
    "description": "Strip leading/trailing whitespace from name column",
    "estimated_rows_affected": 1,
    "reason": "Bob Smith has leading space"
  }
]
```

## Scoring Formula

- **Completeness (30pts)**: 30 - (3 × columns_above_null_threshold) - (1 × columns_with_any_nulls)
- **Consistency (25pts)**: 25 - (5 × schema_violations) - (3 × format_violation_types) - (2 × type_mismatches)
- **Accuracy (25pts)**: 25 - (5 × out_of_range_violations) - (3 × distribution_anomalies) - (2 × volume_anomalies)
- **Timeliness (20pts)**: 20 - (5 if last_modified > 24h) - (3 × staleness_indicators)

Status tiers: Healthy ≥ 80 | Degraded ≥ 50 | Critical < 50

## Memory MCP Usage

- Entity type `"data_baseline"`: stores null_pcts, means, row_count, schema_fingerprint per file
  - Entity name: `"baseline_{filename}"` (e.g., `"baseline_customers.csv"`)
  - Query with: `search_nodes(query="baseline_customers")`

- Entity type `"quality_run"`: stores per-run quality scores for trend tracking
  - Entity name: `"run_{filename}_{timestamp}"` (e.g., `"run_customers.csv_2026-03-31T10:00:00Z"`)
  - Retrieve last 7 runs for trend sparkline

## Auto-Fix Safety Rules

Only apply fixes where `fix_action == "auto-fix"`. Never touch:
- `manual-review` items (ambiguous, needs human judgment)
- `escalate` items (schema drift, data loss risk, security-sensitive)

Conservative application: if affected rows > estimated * 1.1, stop and log warning.
