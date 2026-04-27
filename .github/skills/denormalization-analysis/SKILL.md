---
name: denormalization-analysis
description: >
  Analyzes denormalization patterns in a silver-layer subject area. Use when asked to find
  denormalization issues, update anomalies, or evaluate whether cross-entity attribute
  co-locations are justified.
---

# Denormalization Analysis

For every place where attributes from different logical entities appear together in a single physical table:

## Classification

Classify each instance as:
- **Necessary** — documented performance/read-pattern justification exists
- **Acceptable** — controlled with SCD/audit columns and compensating ETL validation
- **Unnecessary** — causes update anomalies, duplicates canonical data, or collapses identity

## Required Evidence

For each case provide:
- Logical origin (workbook sheet/cell)
- Physical location (CSV row/table & column)
- Missing or present join keys/FKs

## Impact Assessment

- Correctness risk: High / Medium / Low
- Duplication volume
- Backfill complexity: High / Medium / Low

## Remediation Options

1. **Re-normalize**: extract to lookup table
2. **Retain with controls**: add compensating ETL validation — only if documented justification exists

**Default rule**: If no justification is provided, classify as `Unnecessary` and recommend re-normalization.

## Output Format

```markdown
| ID | Tables Involved | Logical Origins | Classification | Correctness Risk | Impact | Recommendation | Confidence |
|---|---|---|---|---|---|---|---|
| DEN-001 | `table_a` | `EntityX.attr`, `EntityY.attr` | Unnecessary | High | Description | Re-normalize | 0.8 |
```
