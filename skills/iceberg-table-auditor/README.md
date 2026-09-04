# Iceberg Table Best-Practices Auditor

## Purpose

Audits Apache Iceberg tables stored on Amazon S3 against AWS best practices and
returns a prioritized, per-table health report with ready-to-run remediation.
It encodes the review a specialist would perform by hand — file sizing and
compaction, snapshot/metadata hygiene, orphan files, deletion safety, S3
layout/throughput, partitioning/sorting, automation/governance, format-version
(V2 vs V3) fit, and statistics/write-time tuning — so any customer can get a
consistent audit on demand.

## Key Capabilities

- Discovers Iceberg tables in the Glue Data Catalog, S3 Tables, or an Athena catalog, and identifies format version and managed-optimizer state.
- Inspects Iceberg metadata (`$files`, `$snapshots`, `$manifests`, `$partitions`, `$history`, `$refs`) to compute file-size distributions, snapshot age, delete-file volume, and per-partition stats.
- Runs a best-practice check catalog across nine domains, each producing a PASS/WARN/FAIL verdict with evidence.
- Produces a scorecard and prioritized, engine-aware remediation (Athena SQL or Spark procedures).
- Is format-version aware: branches logic for Iceberg V2 (delete files) vs V3 (deletion vectors), and flags engine/version incompatibilities (e.g., Athena does not support V3).
- Runs read-only by default (`report` mode); optional `report+remediate` mode runs only safe operations within configured retention, and never mutates a table without explicit opt-in.

## Prerequisites

- An AWS DevOps Agent Space with the target AWS account configured as a cloud source.
- Read access to the target Glue Data Catalog / S3 Tables / Athena catalog and the underlying S3 bucket(s).
- **Data-plane access for full metadata inspection.** Querying Iceberg metadata tables (`$files`, `$snapshots`, etc.) requires SQL execution, which DevOps Agent cannot perform natively. A companion MCP server (Streamable HTTP + SigV4, read-only query allowlist) is required for the full audit. Until that MCP server is registered with the Agent Space, this skill can still run a reduced **control-plane-only mode** (Glue `GetTable`, S3 `ListObjectsV2`, CloudWatch) covering a subset of checks.
- IAM permissions: `glue:GetTable`, `glue:GetDatabase`, `s3:ListBucket`/`s3:GetObject` (read-only), and (for the MCP path) read-only Athena query execution. Most of this is covered by `AIDevOpsAgentAccessPolicy`.

## Limitations

- This is sample code, not intended for production use without additional review and testing. Users should validate in a non-production environment first.
- Does not mutate tables unless the user explicitly opts into `report+remediate` mode, and even then only runs safe, retention-bounded operations.
- Full metadata-table inspection (the core of the audit) depends on a companion MCP server for data-plane access; without it, only the control-plane-only subset of checks runs.
- Athena does not support Iceberg format-version 3; the skill flags this and routes V3 remediation to Spark on EMR or AWS Glue instead.

## Agent Types

Intended for **Chat tasks** and **Evaluation** (proactive operational review) DevOps Agent types.

## Uploading to AWS DevOps Agent

From the `skills/` directory in this repo:

```bash
cd skills
zip -r iceberg-table-auditor.zip iceberg-table-auditor/ -i '*.md' '*.txt' '*.json' '*.yaml' '*.yml' '*.xml' '*.csv' '*.tsv' '*.html' '*.htm' '*.png' '*.jpg' '*.jpeg' '*.gif' '*.svg' '*.webp' '*.pdf' -x '*/.claude/*' '*/scripts/*' '*/README.md' '*/.skilleval.yaml' '*/.skilleval.yml' '*/CHANGELOG.md' '*/evals/*'
```

Then, in the Operator Web App:

1. Navigate to the **Skills** page in your Agent Space.
2. Click **Add skill** → **Upload skill**.
3. Upload `iceberg-table-auditor.zip`.
4. Select agent types: **Chat tasks** and **Evaluation** (or leave **Generic**).
5. Review validation results and click **Upload**.

## How to Use This Skill

In DevOps Agent Chat, use natural language:

- *"Audit my Iceberg table `db.orders` for best practices."*
- *"Why are my Iceberg queries slow on the `sales` namespace?"*
- *"Check my data lake for small files and orphan files."*
- *"Is my Iceberg table ready to upgrade to format version 3?"*

The agent will discover the table(s), run the check catalog, and return a
prioritized findings report with remediation matched to your chosen engine.

## Contributors

- Author: hasandre
- Contributor: kjjanaki

---

⚠️ This skill is sample code, not intended for production use without additional review and testing. Users should validate in a non-production environment first.
