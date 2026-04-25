# {subject_area} Gap Report

> Replace `{subject_area}` with the actual subject area folder name when running an assessment.

## Executive Summary

- Total gaps: _X_ (High: _n_, Medium: _m_, Low: _l_)
- Top 3 recommended immediate actions:
  1. ...
  2. ...
  3. ...

## Gap Table (Summary)

| gap_id | severity | summary | affected_objects | priority |
|---|---:|---|---|---:|
| G-EXAMPLE-001 | High | Example: missing surrogate PK | dbo.customers | P0 |

## Detailed Findings

### G-EXAMPLE-001 — Example: missing surrogate PK
- Severity: High
- Summary: Example: customer surrogate key missing in physical table
- Description:
  Logical model defines a `customer_id` surrogate PK but physical table does not.
- Evidence:
  - File: `{subject_area}_physical_structures.xlsx`, sheet: `customers`, table: `dbo.customers`, column: `customer_number`, cell: B12
- Recommendation:
  1. Add column `customer_id` (BIGINT/UUID as per guideline).
  2. Backfill values using `ROW_NUMBER()` or UUIDs depending on uniqueness of `customer_number`.
  3. Add PK constraint and update downstream ETL to consume `customer_id`.
  4. Validate and then deprecate `customer_number` if appropriate.
- Estimated effort: 24 hours
- Priority: P0
- Owner: Data Modeling / ETL / DBA
- Confidence: 0.92

## Appendix A — Logical → Physical Mapping

| logical_entity | logical_attribute | physical_table | physical_column | status |
|---|---|---|---|---|
| Customer | customer_id | dbo.customers | customer_id | missing (example) |

## Appendix B — Guideline Violations

List violations mapped to guideline documents and headings (cite file and heading).

---

Placeholder instructions:
- Update `generated_on` and `subject_area` in the JSON when used.
- Replace example entries with real findings discovered by the LLM.
- Add remediation plan file if prioritized action items need schedules and owners.
