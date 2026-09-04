<!--
  Iceberg Table Best-Practices Audit — Markdown report template
  =============================================================
  Companion to iceberg-audit-report-template.html. IDENTICAL section structure.
  STRUCTURAL template only — every value is a {{placeholder}} token or a single
  REPEAT example row. Replace tokens with data collected live from the table's
  Iceberg metadata ($files, $snapshots, $manifests, $partitions, $refs), table
  properties, and managed-optimizer / AWS API state. NEVER copy example values
  into a real report.

  This Markdown version is what gets posted directly in the chat response (raw
  HTML pasted into chat renders as inert code). Unlike the HTML companion,
  Markdown renders <>&"' literally — do NOT HTML-escape substituted values here.

  Rows/blocks marked <!-- REPEAT --> are single examples: repeat once per
  returned data row, then delete the comment. Omit any domain with zero findings.
  Status convention: ✅ PASS · ⚠️ WARN · ❌ FAIL · ℹ️ INFO
-->

# 🧊 Iceberg Table Best-Practices Audit

**{{account_or_customer_label}}** · {{report_date}}
**Table:** `{{table_fqn}}` · **Format:** v{{format_version}} · **Catalog:** {{catalog_type}} · **Engine:** {{engine}}

---

## Executive Summary

| Pass | Warn | Fail | Info |
|:----:|:----:|:----:|:----:|
| ✅ {{pass_count}} | ⚠️ {{warn_count}} | ❌ {{fail_count}} | ℹ️ {{info_count}} |

**Overall Health: {{overall_health_emoji_and_label}}**

{{executive_summary_narrative}}

A total of **{{total_findings}}** checks were evaluated across eight domains: **{{fail_count}} critical (FAIL)**, **{{warn_count}} warnings**, **{{pass_count}} passing**, and {{info_count}} informational.

### Critical Findings Requiring Immediate Attention

- **{{critical_finding_category}}: {{critical_finding_title}}:** {{critical_finding_detail}}
<!-- REPEAT per FAIL-severity finding -->

### Top Optimization Opportunities

- **{{opportunity_category}}: {{opportunity_title}}:** {{opportunity_detail}}
<!-- REPEAT per top WARN-severity opportunity -->

### Areas of Concern

- **{{area_category}}**: {{area_fail_count}} critical, {{area_warn_count}} warnings
<!-- REPEAT per finding domain -->

---

## 1. Table Overview

| Property | Value |
|---|---|
| Table (fully qualified) | `{{table_fqn}}` |
| Catalog Type | {{catalog_type}} |
| Format Version | v{{format_version}} |
| Total Data Files | {{data_file_count}} |
| Total Data Size | {{total_data_size}} |
| Median / Avg File Size | {{median_file_size}} / {{avg_file_size}} |
| Partition Spec | {{partition_spec}} |
| Partition Count | {{partition_count}} |
| Snapshot Count | {{snapshot_count}} (oldest {{oldest_snapshot_age}}) |
| Delete Files / Deletion Vectors | {{delete_artifact_summary}} |
| Managed Optimizer | {{managed_optimizer_state}} |
| Sort Order | {{sort_order}} |

---

## 2. All Findings

| Domain | Check | Status | Detail | Recommendation |
|---|---|:---:|---|---|
| File Sizing & Compaction | {{check_id}}: {{check_name}} | ❌ FAIL | {{check_detail}} | {{check_recommendation}} |
| Snapshot & Metadata | {{check_id}}: {{check_name}} | ⚠️ WARN | {{check_detail}} | {{check_recommendation}} |
<!-- REPEAT one row per finding, grouped by domain (A–H) -->

---

## 3. File Sizing & Compaction

| Metric | Value |
|---|---|
| Avg File Size | {{avg_file_size}} |
| Target Size | {{target_file_size}} |
| % Below Target | {{small_file_pct}}% |
| Delete Files / DVs | {{delete_artifact_count}} |

### Per-Partition File Sizing

| Partition | Files | Avg Size | Below Target | Status | Detail |
|---|---:|---|---:|:---:|---|
| {{partition_value}} | {{partition_file_count}} | {{partition_avg_size}} | {{partition_below_target}} | {{severity_label}} | {{partition_detail}} |
<!-- REPEAT per partition (or top-N worst partitions) -->

> **What's happening:** {{compaction_issue_explanation}}
> **Impact:** {{compaction_impact}}
> **Fix:** {{compaction_fix}}

**Remediation:**

```sql
{{compaction_remediation_sql}}
```

---

## 4. Snapshot & Metadata Hygiene

| Metric | Value |
|---|---|
| Snapshots | {{snapshot_count}} |
| Oldest Snapshot | {{oldest_snapshot_age}} |
| Metadata Files | {{metadata_file_count}} |
| Manifests | {{manifest_count}} |

| Check | Status | Detail | Recommendation |
|---|:---:|---|---|
| {{check_id}}: {{check_name}} | {{severity_label}} | {{check_detail}} | {{check_recommendation}} |
<!-- REPEAT per snapshot/metadata check -->

> ⚠️ **Caution:** Snapshot expiration is a hard delete. Time-travel and rollback to expired snapshots are lost. Confirm retention requirements before expiring.

**Remediation:**

```sql
{{snapshot_remediation_sql}}
```

---

## 5. Orphan Files

{{orphan_summary_narrative}}

| Check | Status | Detail | Recommendation |
|---|:---:|---|---|
| C1: Orphan (unreferenced) files | {{severity_label}} | {{orphan_detail}} | {{orphan_recommendation}} |

> **Safety:** Always run orphan removal with `dry_run => true` first. Never set retention shorter than the longest in-flight write, or in-progress files may be deleted and corrupt the table (default retention 3 days).

**Remediation:**

```sql
{{orphan_remediation_sql}}
```

---

## 6. Deletion Safety

| Check | Status | Detail | Recommendation |
|---|:---:|---|---|
| D1: Non-Iceberg deletes / S3 Lifecycle on table data | {{severity_label}} | {{deletion_safety_detail}} | {{deletion_safety_recommendation}} |

> ❌ **Critical guardrail:** Deleting or overwriting an Iceberg table's underlying S3 objects with non-Iceberg methods (CLI, SDK, Boto3) or S3 Lifecycle expiration on data/metadata prefixes corrupts the table and breaks queries. Release storage only through Iceberg-native operations.

---

## 7. S3 Layout & Throughput

| Check | Status | Detail | Recommendation |
|---|:---:|---|---|
| E1: write.distribution-mode | {{severity_label}} | {{distribution_mode_detail}} | {{distribution_mode_recommendation}} |
| E2: Prefix hot-spotting / HTTP 503 | {{severity_label}} | {{prefix_detail}} | {{prefix_recommendation}} |
| E3: Storage class | {{severity_label}} | {{storage_class_detail}} | {{storage_class_recommendation}} |

---

## 8. Partitioning & Sorting

| Check | Status | Detail | Recommendation |
|---|:---:|---|---|
| F1: Sort order vs query patterns | {{severity_label}} | {{sort_detail}} | {{sort_recommendation}} |
| F2: Partition granularity | {{severity_label}} | {{partition_granularity_detail}} | {{partition_granularity_recommendation}} |

---

## 9. Automation & Governance

| Check | Status | Detail | Recommendation |
|---|:---:|---|---|
| G1: Managed optimization enabled | {{severity_label}} | {{managed_opt_detail}} | {{managed_opt_recommendation}} |
| G2: Optimizer health | {{severity_label}} | {{optimizer_health_detail}} | {{optimizer_health_recommendation}} |
| G3: Cross-Region replication | {{severity_label}} | {{crr_detail}} | {{crr_recommendation}} |

---

## 10. Format Version & Feature Fit (V2 / V3)

| Check | Status | Detail | Recommendation |
|---|:---:|---|---|
| H1: Version-appropriate delete handling | {{severity_label}} | {{h1_detail}} | {{h1_recommendation}} |
| H2: Post-upgrade compaction | {{severity_label}} | {{h2_detail}} | {{h2_recommendation}} |
| H3: Engine / version compatibility | {{severity_label}} | {{h3_detail}} | {{h3_recommendation}} |
| H4: Variant shredding compatibility | {{severity_label}} | {{h4_detail}} | {{h4_recommendation}} |
| H5: V3 modernization opportunity | ℹ️ INFO | {{h5_detail}} | {{h5_recommendation}} |

> ℹ️ **Note:** {{format_version_note}}

---

## 11. Statistics & Write-Time Tuning

| Check | Status | Detail | Recommendation |
|---|:---:|---|---|
| I1: Column statistics (CBO) | {{severity_label}} | {{i1_detail}} | {{i1_recommendation}} |
| I2: Wide-table metrics cap (>100 cols) | {{severity_label}} | {{i2_detail}} | {{i2_recommendation}} |
| I3: Bloom filters for equality predicates | {{severity_label}} | {{i3_detail}} | {{i3_recommendation}} |
| I4: Write-time small-file / manifest tuning | {{severity_label}} | {{i4_detail}} | {{i4_recommendation}} |

**Remediation:**

```sql
{{statistics_remediation_sql}}
```

---

## 12. Prioritized Recommendations

1. **[FAIL] {{recommendation_1_title}}** — {{recommendation_1_detail}}
   💡 {{recommendation_1_fix}}
<!-- REPEAT one numbered item per recommendation, ordered by severity (FAIL > WARN > INFO) -->

---

*Generated by the Iceberg Table Auditor · {{report_date}} · A downloadable HTML version of this report is also available.*
