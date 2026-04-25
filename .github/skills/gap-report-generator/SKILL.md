---
name: gap-report-generator
description: >
   Generates the final structured gap assessment report for a silver-layer subject area.
   Use when asked to generate, write, or produce the gap report output file after assessment
   tasks are complete.
---

# Gap Report Generator

After all assessment tasks are complete, generate the final report at:
`sa/<subject_area>/output/<subject_area>_gap_report.md`

## Report Structure

The report **must** follow this exact structure. Do not omit any section — write "No issues found" where clean.

### Sections

1. **Header** — Subject area metadata table (name, schema, date, overall state, entities assessed, total findings, confidence)

2. **Executive Summary**
   - 1A: Management slide (one-line health, 3–5 bullet risks, Top 5 Actions table, 2 KPIs, decision recommendations)
   - Technical paragraph summary

3. **Subject Area Architecture Assessment** (2.1–2.6)
   - 2.1 Domain Scoping & BIAN Alignment
   - 2.2 Identity Strategy
   - 2.3 SCD Strategy
   - 2.4 Cross-Entity Relationships & Cardinality
   - 2.5 ETL Mapping & Lineage Completeness
   - 2.6 Major Design Gaps table

4. **Entity Inventory** — Summary table of all entities with state, criticality, mapped tables, unmapped attrs, finding counts, confidence

5. **Per-Entity Assessments** — Section 4.x for each entity with:
   - State, criticality, confidence, physical tables
   - Industry fitness narrative
   - Attribute review table
   - Metadata columns checklist
   - Findings (each as a sub-section with full detail)

6. **Denormalization Register** — Table + justification notes

7. **Guideline Compliance Summary** — Rule-by-rule pass/fail table

8. **Remediation Plan** — Prioritized action list + suggested sprint schedule

9. **Appendix**
   - A: Logical → Physical mapping summary
   - B: Guideline citations with full rule text
   - C: Industry standard references (BIAN/IFW/IFRS 9/BCBS 239)

## State Indicators

- 🔴 Incorrect — correctness, regulatory, or data-loss risk
- 🟡 Partial — data quality, FK integrity, type mismatch
- 🟢 OK — compliant

## Scoring

- **Confidence**: 0.0–1.0 per entity and finding
- **Priority**: P0 (immediate) / P1 (next sprint) / P2 (backlog)
- **Criticality**: High / Medium / Low
