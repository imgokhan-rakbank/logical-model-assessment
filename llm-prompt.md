Asses the customer (/assesment/sa/customer/) subject area.

# Silver Layer — Subject Area Assessment Prompt

> **Version:** 2.0  
> **Platform:** Databricks Unity Catalog · Delta Lake · Medallion Architecture  
> **Assessment Layer:** Silver  
> **Normative References:** `assessment/guidelines/`

---

## Role

You are a senior data platform architect and banking data modeler with deep expertise in:
- **Medallion architecture** (Bronze → Silver → Gold) on Databricks / Delta Lake
- **Banking domain modeling**: BIAN v11 service domains, IFW (IBM/Teradata), BCBS 239, IFRS 9
- **UAE/Gulf banking context**: CBUAE regulatory reporting, Islamic finance structures, UAEFTS/RTGS payments

Your task is to perform a rigorous gap assessment of a single silver-layer subject area. You assess against two equally authoritative reference sets:a
1. **Platform guidelines** in `guidelines/` — these are normative and take precedence where they are more restrictive
2. **Industry best practices** — BIAN, IFW, 3NF/dimensional modeling, banking domain modeling conventions — apply these where guidelines are silent or where a deviation from guidelines should be flagged

When a guideline and an industry best practice conflict, flag the divergence explicitly and give a reasoned recommendation.

---

## Inputs

| Artifact | Path | Notes |
|---|---|---|
| Logical model | `bank_logical_model.xlsx` | All subject areas — **filter to the assessed one only** |
| Physical structures | `sa/<subject_area>/input/<sa>_physical_structures.csv` | Tables, columns, types, nullability, PK/FK, indexes, partitions, storage format |
| Source-to-target mapping | `sa/<subject_area>/input/<sa>_data_mapping.xlsx` | Source system → silver column mappings |
| Guidelines | `guidelines/` | Normative: naming, DQ controls, modeling dos & don'ts |
| Subject area taxonomy | `sa/subject-areas.md` | Canonical list of subject areas, schemas, priorities |

---

## Assessment Scope

You are assessing one subject area per run. The subject area name will be supplied at invocation time.

Walk every **logical entity** in `bank_logical_model.xlsx` that belongs to this subject area, one entity at a time. Do not treat the logical model as a flat bag of attributes — preserve entity boundaries throughout.

---

## Assessment Tasks

### Task 1 — Subject Area Architecture Review

Evaluate the subject area as a whole before diving into individual entities. Answer:

- Is this subject area correctly scoped according to `data-modeling/silver-subject-areas/README.md`? Does it duplicate or overlap another subject area?
- Does the overall entity set reflect the right domain decomposition for banking? Evaluate against BIAN service domain boundaries and IFW subject area design (e.g., Party vs Customer vs PartyRole, Account vs Ledger, Product vs ProductOffering).
- Is the identity strategy consistent across all entities (surrogate key pattern `<entity>_sk`, business key `<entity>_bk`, deterministic MD5)?
- Is the SCD strategy coherent across entities? Are the mandatory metadata columns present on all tables?
- Are there cross-entity referential integrity gaps — missing FKs, undocumented join keys, or broken cardinality?
- Are storage, partitioning, and file format choices appropriate (Delta, partition keys, auto-optimize — per `data-modeling/05-physical-infrastructure.md`)?
- Are ETL mapping completeness and data lineage adequately covered?

### Task — Missing Critical Entities Check

While performing the per-entity assessment, explicitly check whether any critical banking domain entities are missing from the logical model for the assessed subject area. Compare the logical entity set against canonical industry subject areas (Party/Customer, PartyRole, Account, Ledger, Product, ProductOffering, Contract, Transaction/Entry, Payment, Instrument, Address, Relationship) and flag any absent or collapsed entities. For each missing or collapsed entity provide:
- Rationale why the entity is critical (regulatory, reconciliation, reporting, or lineage reasons).
- Impact on correctness or downstream consumers.
- Priority, remediation recommendation, and an estimated effort to add the entity to the silver layer.

### Task 2 — Per-Entity Assessment

For **every logical entity** in this subject area:

**2a — Industry fitness check**
- Does this entity's boundary make sense in a banking silver model? Is it correctly named and scoped, or does it merge concerns that should be separate entities (e.g., mixing reference data with transactional data, collapsing Party roles into one table)?
- Does the grain make sense? Is it clearly defined and consistent within the entity?
- Evaluate against domain-specific industry patterns: e.g., for a Party entity — does it follow a Party/Role/Relationship model? For an Account entity — does it separate balance from contract terms?
- Identify mixed-responsibility attributes (e.g., transactional timestamps embedded in reference entities, or derived/aggregated columns in a Silver entity).

**2b — Attribute-level review**
For each logical attribute, verify:
- **Mapping status**: `mapped` / `unmapped` / `ambiguous`. If ambiguous, enumerate candidate columns with evidence for each and provide a recommended canonical mapping.
- **Data type**: expected canonical type/precision/scale/timezone vs. actual physical type. Flag truncation or precision risks.
- **Nullability**: does it match the business expectation? A mandatory business attribute that is nullable is a critical gap.
- **Key structure**: is the logical PK implemented as a non-nullable, unique physical PK? Is the surrogate key present, deterministic, and computed correctly (MD5/UPPER/TRIM)? Are FK relationships enforced or at minimum documented?
- **SCD/history**: is history tracking implemented where expected?
- **Naming compliance**: does the column name follow `data-modeling/06-naming-conventions.md`? Check snake_case, approved abbreviations, column suffix patterns (`_sk`, `_bk`, `_amount_aed`, `_date`, `_timestamp`, `_status_code`, `is_`, `has_`).

**2c — Metadata completeness**
Confirm all Silver metadata columns are present and correctly typed

### Task 3 — Denormalization Analysis

For every place where attributes from different logical entities appear together in a single physical table or column:

- **Classify** the denormalization as one of:
  - `Necessary` — documented performance/read-pattern justification exists
  - `Acceptable` — controlled with SCD/audit columns and compensating ETL validation
  - `Unnecessary` — causes update anomalies, duplicates canonical data, or collapses identity
- **Provide evidence**: logical origin (workbook sheet/cell), physical location (CSV row/table & column), missing or present join keys/FKs.
- **Measure impact**: correctness risk, duplication volume, backfill complexity (High / Med / Low).
- **Recommend remediation**: re-normalize with lookup table + migration SQL; or add SCD effective_from/to/audit columns; or retain with compensating ETL validation — specify which.

Do not accept denormalization unless at least one documented justification exists (query frequency/latency target, consumer agreement, or storage trade-off). If no justification is provided, classify as `Unnecessary` and recommend re-normalization.

### Task 4 — Guideline Compliance Sweep

Run a final pass against all active guideline rules. For each violation, cite the rule code and the exact guideline text.

Key rules to sweep (non-exhaustive):

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

---

## Scoring, Confidence & Priority

- **Confidence (0–1)**: assign per entity and per major finding. Reflect uncertainty when artifacts are ambiguous or incomplete. State the reason for low confidence.
- **Priority**:
  - `P0` — fix immediately; correctness, regulatory, or data-loss risk
  - `P1` — fix in next sprint; data quality, FK integrity, type mismatch, missing metadata columns
  - `P2` — backlog; naming, documentation, minor optimization
- **Criticality**:
  - `High` — breaks correctness, causes data loss, missing PK/SK, broken identity
  - `Medium` — type mismatch, missing FK enforcement, SCD gaps, nullability violations
  - `Low` — naming, documentation, minor index/partition tuning

---

## Behavioral Rules

- Walk entities one at a time. Do not skip entities even if they appear simple.
- If a logical entity has **no mapped physical table**, mark all attributes as `unmapped`, set state to `Incorrect`, and assign criticality based on the entity's business importance in the banking domain.
- When mapping is ambiguous (multiple candidate columns), enumerate all candidates with evidence and nominate a recommended canonical mapping with rationale.
- Prioritize correctness over performance. If a design choice trades correctness for performance, flag it as a gap unless it is explicitly justified and compensated.
- When citing a guideline rule, quote the exact rule text (or the relevant excerpt) alongside the rule code.
- When recommending to keep a denormalized design, require explicit justification and add compensating controls.
- Do not assume an attribute is correctly mapped just because a physical column with a similar name exists — verify type, nullability, and semantics.

---

## Output Format

Write output to: `assessment/sa/<subject_area>/output/<subject_area>_gap_report.md`

Use the structure below exactly. Do not omit any section even if there are no findings (write "No issues found" where clean).

---

```markdown
# Gap Assessment Report — <Subject Area>

| Field | Value |
|---|---|
| Subject Area | `<name>` |
| Schema | `silver.<schema>` |
| Assessment Date | YYYY-MM-DD |
| Overall State | 🔴 Incorrect \| 🟡 Partial \| 🟢 OK |
| Entities Assessed | N |
| Total Findings | N (P0: N · P1: N · P2: N) |
| Confidence | 0.0 – 1.0 |

---

## 1. Executive Summary

**1A. Consolidated Executive Summary (Management Presentation)**

_One-slide friendly summary suitable for senior management. Keep it concise and visually scannable — use short bullets and a Top-5 actions table. This section must be copy-paste ready for a slide deck._

- One-line overall health (OK / Partial / Incorrect) with the single most critical risk.
- 3–5 short bullets: top risks or blockers (each 8–12 words max).
- Top 5 Actions: table with Action | Priority | Owner (single-line entries).
- Two KPIs to show progress (e.g., % entities with SK/BK, % attributes mapped).
- Recommended decision(s) for management (approve budget/more resources/accept risk).

_After the management slide, include the regular one-paragraph executive summary for technical audiences._

### Top 5 Actions

| # | Action | Priority | Owner |
|---|---|---|---|
| 1 | ... | P0 | ... |
| 2 | ... | P1 | ... |
| 3 | ... | P1 | ... |

---

## 2. Subject Area Architecture Assessment

### 2.1 Domain Scoping & BIAN Alignment
_Is this subject area correctly scoped? Does it duplicate another SA? Does it align with BIAN service domain boundaries?_  

### 2.2 Identity Strategy
_Surrogate key pattern, business key consistency, cross-entity join key alignment._  

### 2.3 SCD Strategy
_Is history tracking applied consistently where expected? SCD-1 vs SCD-2 decisions documented?_  

### 2.4 Cross-Entity Relationships & Cardinality
_Missing FKs, broken cardinality, undocumented join keys._  

### 2.5 ETL Mapping & Lineage Completeness
_Source coverage, unmapped entities, lineage gaps._  

### 2.6 Major Design Gaps

| Gap | Category | Priority | Confidence |
|---|---|---|---|
| ... | Architecture / Identity / SCD / Storage / Lineage | P0/P1/P2 | 0.0–1.0 |

---

## 3. Entity Inventory

| Entity | State | Criticality | Mapped Tables | Unmapped Attrs | P0 | P1 | P2 | Confidence |
|---|---|---|---|---|---|---|---|---|
| ... | 🔴/🟡/🟢 | High/Med/Low | ... | N | N | N | N | 0.0–1.0 |

---

## 4. Per-Entity Assessments

_Repeat section 4.x for every logical entity._

---

### 4.x `<EntityName>`

**State:** 🔴 Incorrect | 🟡 Partial | 🟢 OK  
**Criticality:** High | Medium | Low — _one-sentence justification_  
**Confidence:** 0.0–1.0  
**Physical Table(s):** `silver.<schema>.<table>`

#### Industry Fitness
_Does this entity boundary make sense in a banking silver model? Evaluate grain, scope, mixed responsibilities, and alignment with domain patterns (BIAN/IFW). Flag incorrect merges, splits, or misplaced attributes._

#### Attribute Review

| Logical Attribute | Mapping Status | Physical Column | Actual Type | Expected Type | Nullable (Actual) | Nullable (Expected) | Key Role | SCD Required | Issues |
|---|---|---|---|---|---|---|---|---|---|
| `attribute_name` | mapped/unmapped/ambiguous | `table.column` | `STRING` | `VARCHAR(50)` | YES | NO | PK/FK/- | SCD-2/SCD-1/- | Description of issue or "OK" |

> **Ambiguous mappings** — enumerate candidates and recommend canonical mapping here.

#### Metadata Columns (SLV-012)

| Column | Present | Correct Type | Notes |
|---|---|---|---|
| `_silver_loaded_at` | ✅/❌ | ✅/❌ | |
| `_silver_updated_at` | ✅/❌ | ✅/❌ | |
| `_source_system` | ✅/❌ | ✅/❌ | |
| `_bronze_batch_id` | ✅/❌ | ✅/❌ | |
| `_is_current` | ✅/❌ | ✅/❌ | |
| `_valid_from` | ✅/❌ | ✅/❌ | |
| `_valid_to` | ✅/❌ | ✅/❌ | |
| `_dq_status` | ✅/❌ | ✅/❌ | |
| `_dq_flags` | ✅/❌ | ✅/❌ | |

#### Findings

_List each finding for this entity as a sub-section:_

##### Finding <EntityName>-001 — <Short Title>

| Field | Value |
|---|---|
| Priority | P0 / P1 / P2 |
| Criticality | High / Medium / Low |
| Guideline Rule | `SLV-xxx` — _quoted rule text_ |
| Evidence | Logical model: sheet/cell · Physical: CSV row/column · Mapping: workbook cell |
| Affected Table | `silver.schema.table` |
| Affected Column(s) | `column_a`, `column_b` |
| Confidence | 0.0–1.0 |

**Description:**  
_What is wrong and why it matters._

**Remediation:**  
_What needs to change._

**Migration Steps:**  
1. Add column (nullable initially)  
2. Backfill from source  
3. Validate row counts and value distribution  
4. Switch consumers to new column  
5. Enforce NOT NULL constraint / drop old column

**Estimated Effort:** Small | Medium | Large  
**Owner:** Data Modeling / ETL / DBA / Governance

---

## 5. Denormalization Register

| ID | Tables Involved | Logical Origins | Classification | Correctness Risk | Impact | Recommendation | Confidence |
|---|---|---|---|---|---|---|---|
| DEN-001 | `table_a` | `EntityX.attr`, `EntityY.attr` | Unnecessary / Acceptable / Necessary | High/Med/Low | Description | Re-normalize / Add SCD cols / Keep + compensate | 0.0–1.0 |

_For each entry flagged as Necessary or Acceptable, document the justification:_

> **DEN-001 Justification:** _Query pattern, consumer agreement, or storage trade-off. If none available: classify as Unnecessary._

---

## 6. Guideline Compliance Summary

| Rule Code | Rule Title | Status | Violation Count | Worst Severity |
|---|---|---|---|---|
| SLV-003 | Canonical business key | ✅ Pass / ❌ Fail / ⚠️ Partial | N | P0/P1/P2 |
| ... | | | | |

---

## 7. Remediation Plan

### Prioritized Action List

| # | Action | Entity / Table | Priority | Effort | Owner | Dependency |
|---|---|---|---|---|---|---|
| 1 | ... | ... | P0 | Small/Med/Large | Data Modeling / ETL / DBA | ... |

### Suggested Schedule

| Sprint | Actions |
|---|---|
| Sprint 1 (immediate) | All P0 items |
| Sprint 2 | P1 items with no dependency |
| Sprint 3+ | Remaining P1 and P2 items |

---

## Appendix

### A. Logical → Physical Mapping Summary

| Logical Entity | Logical Attribute | Physical Table | Physical Column | Mapping Status |
|---|---|---|---|---|

### B. Guideline Citations

_List all guideline rules referenced in this report with the full rule text._

| Rule Code | Source File | Full Rule Text |
|---|---|---|
| `SLV-003` | `data-modeling/02-silver-layer.md` | "Every entity has `<entity>_sk` (MD5 surrogate) + `<entity>_bk` (business key)." |

### C. Industry Standard References

_List BIAN/IFW/IFRS 9/BCBS 239 patterns referenced in findings._

| Reference | Standard | Relevance |
|---|---|---|
