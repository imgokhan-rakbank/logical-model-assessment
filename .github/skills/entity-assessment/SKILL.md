---
name: entity-assessment
description: >
  Performs a detailed per-entity assessment of a silver-layer logical model entity including
  industry fitness, attribute-level review, metadata completeness, and finding generation.
  Use when asked to assess, audit, or review a specific entity or all entities in a subject area.
allowed-tools: shell
---

# Per-Entity Assessment

For each logical entity in the subject area, perform the following checks:

## Industry Fitness Check

- Does the entity boundary make sense in a banking silver model?
- Is the entity correctly named and scoped?
- Does it merge concerns that should be separate (e.g., mixing reference with transactional data)?
- Is the grain clearly defined and consistent?
- Evaluate against domain patterns:
  - **Party**: Party/Role/Relationship model
  - **Account**: separate balance from contract terms
  - **Product**: catalog vs instance
  - **Transaction**: double-entry postings

## Attribute-Level Review

For each logical attribute, check:

| Check | Detail |
|---|---|
| **Mapping status** | `mapped` / `unmapped` / `ambiguous` — if ambiguous, list candidates with evidence |
| **Data type** | Expected canonical type vs actual physical type; flag truncation/precision risks |
| **Nullability** | Must match business expectation; mandatory attributes that are nullable = critical gap |
| **Key structure** | PK must be non-nullable + unique; SK present and deterministic; FK documented |
| **SCD/history** | History tracking where expected |
| **Naming** | snake_case, approved abbreviations, correct suffix patterns (`_sk`, `_bk`, `_amount_aed`, `_date`, `_timestamp`, `_status_code`, `is_`, `has_`) |

## Metadata Completeness (SLV-008)

Every Silver entity must have these columns with correct types:

| Column | Expected Type |
|---|---|
| `_silver_loaded_at` | TIMESTAMP |
| `_silver_updated_at` | TIMESTAMP |
| `_source_system` | STRING |
| `_bronze_batch_id` | STRING |
| `_is_current` | BOOLEAN |
| `_valid_from` | TIMESTAMP |
| `_valid_to` | TIMESTAMP |
| `_dq_status` | STRING |
| `_dq_flags` | STRING / ARRAY |

## Finding Format

For each issue found, generate:

```
### Finding <EntityName>-NNN — <Short Title>

| Field | Value |
|---|---|
| Priority | P0 / P1 / P2 |
| Criticality | High / Medium / Low |
| Guideline Rule | `SLV-xxx` — "quoted rule text" |
| Evidence | Logical model: location · Physical: CSV row · Mapping: cell |
| Affected Table | `silver.schema.table` |
| Affected Column(s) | `col_a`, `col_b` |
| Confidence | 0.0–1.0 |

**Description:** What is wrong and why it matters.
**Remediation:** What needs to change.
**Estimated Effort:** Small / Medium / Large
**Owner:** Data Modeling / ETL / DBA / Governance
```
