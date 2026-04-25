---
name: guideline-compliance-sweep
description: >
  Runs a comprehensive guideline compliance sweep against all active rules for a silver-layer
  subject area. Use when asked to check naming conventions, DQ controls, SCD rules, or overall
  guideline compliance.
allowed-tools: shell
---

# Guideline Compliance Sweep

Run a final pass of the subject area against all guideline rules from `guidelines/`. For each violation, cite the rule code and quote the exact guideline text.

## Steps

1. Read all guideline files in `guidelines/`.
2. Read the physical structures CSV for the subject area.
3. For each rule below (and any others found in guidelines), check every entity and attribute.

## Key Rules

| Rule | Check |
|---|---|
| SLV-001 / SLV-002 | `<entity>_sk` (MD5) + `<entity>_bk` on every entity |
| SLV-003 | SCD-2 default; SCD-1 only with steward approval |
| SLV-004 | No source-system codes in entity columns; mapped via `silver.reference.code_mapping` |
| SLV-005 | DQ-gated writes; quarantine table present |
| SLV-006 | 3NF within subject area; no derived values or aggregations |
| SLV-007 | No pre-computed metrics in Silver |
| SLV-008 | All metadata columns present, correctly typed |
| SLV-009 | All timestamps in UTC |
| SLV-010 | Monetary amounts in smallest unit (fils for AED); `amount_orig` + `currency_code` carried |
| NAM-001 | snake_case, lowercase, no reserved words |
| NAM-003 | Table naming pattern: `<entity>` for Silver entity tables |
| NAM-005 | No SQL reserved words as identifiers |

## Output Format

```markdown
| Rule Code | Rule Title | Status | Violation Count | Worst Severity |
|---|---|---|---|---|
| SLV-001 | Surrogate key | ✅ Pass / ❌ Fail / ⚠️ Partial | N | P0/P1/P2 |
```

For each violation, provide:
- Rule code and quoted rule text
- Affected entity/table/column
- Evidence (physical CSV row, logical model location)
- Recommended fix
