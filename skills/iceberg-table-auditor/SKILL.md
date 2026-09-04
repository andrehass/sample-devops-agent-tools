---
name: iceberg-table-auditor
description: >
  Audits Apache Iceberg tables stored in Amazon S3 against AWS best practices
  for file sizing/compaction, snapshot & metadata hygiene, orphan files, S3
  layout/throughput, partitioning & sorting, optimization automation,
  format-version fit, and statistics/write-time tuning. Produces a per-table
  findings report (PASS/WARN/FAIL) with remediation SQL and CLI. Use when a
  user asks to review, audit, health-check, or optimize an Iceberg
  table/lakehouse on S3, or mentions small files, slow Iceberg queries,
  snapshot/metadata bloat, orphan files, or S3 503/throttling on a data lake.
metadata:
  author: hasandre
  version: "1.0.0"
---

# Iceberg Table Best-Practices Auditor

> **Status:** scaffold. Check-by-check instructions are being built incrementally,
> domain by domain, with review at each step before moving to the next
> (see the build methodology in the project spec). This file will be filled in
> as each domain (A–I) is implemented and confirmed accurate.

## When to Use

Activate this skill when the user asks to:
- Review, audit, or health-check an Apache Iceberg table or lakehouse on Amazon S3
- Investigate why Iceberg queries are slow
- Diagnose small files, snapshot/metadata bloat, or orphan files on an Iceberg table
- Optimize an Iceberg table's layout, partitioning, or compaction settings
- Assess Iceberg V2 vs V3 format-version fit or upgrade readiness

## Inputs

| Parameter | Required | Description |
|---|---|---|
| `catalog` | yes | Glue catalog name, S3 Tables bucket, or Athena catalog. |
| `database` / `namespace` | yes | Schema/namespace containing the table(s). |
| `table` | no | Specific table. If omitted, audit all tables in the namespace. |
| `engine` | no | `athena` (default) \| `spark_emr` \| `spark_glue`. |
| `region` | no | AWS region; default from environment. |
| `mode` | no | `report` (default) \| `report+remediate`. |
| `thresholds` | no | Override defaults in `references/thresholds.md`. |

## Workflow

1. **Discovery & preflight** — resolve the table(s) in the catalog, confirm Iceberg format version, read table properties, detect managed optimizer state.
2. **Metadata inspection** — query Iceberg metadata tables (`$files`, `$snapshots`, `$manifests`, `$partitions`, `$history`, `$refs`).
3. **Run the check catalog** — see `references/check-catalog.md` for the full rubric across domains A–I.
4. **Score and prioritize** — aggregate PASS/WARN/FAIL into a scorecard, rank highest-impact fixes first.
5. **Generate remediation** — engine-aware SQL/CLI per finding, from `references/remediation-cookbook.md`.
6. **Report** — render findings as Markdown (human) and JSON (machine); see `assets/iceberg-audit-report-template.md`.
7. **(Optional) Safe remediation** — only in `report+remediate` mode; see the constraints in `references/thresholds.md`.

Detailed detection logic and thresholds for each check will be added to `references/check-catalog.md` and `references/thresholds.md` as they are built and reviewed, one domain at a time.

## Output

- Never mutate tables in `report` mode (the default).
- `report+remediate` runs only safe operations (bin-pack compaction, orphan/snapshot cleanup within retention), prints every statement before running, and never expires snapshots below retention, deletes by non-Iceberg means, or touches lifecycle rules.
- Report template: `assets/iceberg-audit-report-template.md` (Markdown) with an HTML companion.
