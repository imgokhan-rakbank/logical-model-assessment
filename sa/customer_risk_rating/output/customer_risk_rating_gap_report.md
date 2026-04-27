# Silver-Layer Gap Assessment Report
## Subject Area: Customer Risk Rating & Scoring
**Report Date:** 2026-04-27  
**Assessed By:** Automated Silver-Layer Gap Assessment (Senior Data Platform Architect)  
**Repository:** imgokhan-rakbank/logical-model-assessment  
**Report Path:** `sa/customer_risk_rating/output/customer_risk_rating_gap_report.md`

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Subject Area Architecture Assessment](#2-subject-area-architecture-assessment)
3. [Entity Inventory](#3-entity-inventory)
4. [Per-Entity Assessments](#4-per-entity-assessments)
5. [Denormalization Register](#5-denormalization-register)
6. [Guideline Compliance Summary](#6-guideline-compliance-summary)
7. [Remediation Plan](#7-remediation-plan)
8. [Appendix](#8-appendix)

---

## 1. Executive Summary

### 1.1 Management Slide

| Dimension | Status | Summary |
|---|---|---|
| **Overall Health** | 🔴 Critical | Subject area has 19 P0/P1 findings across architecture, modeling, and compliance dimensions |
| **Entity Coverage** | 🟡 Partial | 7 of 8 logical entities have a physical counterpart; 2 core entities have empty tables (0 bytes) |
| **Schema Alignment** | 🔴 Non-Compliant | Physical schema is `silver_dev.customer_riskrating_scoring`; canonical schema per SA registry is `silver.risk` |
| **Identity Strategy** | 🔴 Missing | No `<entity>_sk` (MD5 surrogate) or `<entity>_bk` (business key) pattern on any entity |
| **SCD Strategy** | 🔴 Non-Compliant | SCD-2 columns (`effective_from`, `effective_to`, `is_current`) absent from all entities; non-standard `Business Start Date` / `Business End Date` used instead |
| **3NF Compliance** | 🔴 Critical | 28 of 39 physical tables are Bronze-like staging replicas of source tables (Bureauone), not normalized Silver entities |
| **Metadata Columns** | 🔴 Non-Compliant | `run_id`, `source_ingestion_date`, `is_current`, `effective_from`, `effective_to` absent; `IsLogicallyDeleted` violates snake_case |
| **Naming Conventions** | 🔴 Non-Compliant | PascalCase names (`IsLogicallyDeleted`, `Business Start Date`), no `_sk`/`_bk` suffixes, table naming conflicts with entity naming standards |
| **Data Type Precision** | 🔴 Missing | All 109 logical attributes typed as STRING regardless of semantics (dates, scores, amounts) |
| **Mapping Completeness** | 🟡 Partial | 34 of 93 mapped attributes have `Not Available` or null status; 0 attributes have data-type spec in mapping |
| **Scoping Compliance** | 🔴 Critical | `Loan_Account` entity (141 attributes) included in CRR logical model — belongs to `silver.lending` |
| **DQ Controls** | 🔴 Missing | No quarantine table, no DQ-gate evidence, `data_quality_status`/`dq_issues` columns absent |
| **Aggregations in Silver** | 🔴 Violation | `reference_aggregated_pipelines` and `reference_aggregated_individualcontracts` are pre-aggregated tables — prohibited in Silver |

**Confidence Score:** 0.91 (high — artifacts are complete and findings are directly evidenced)

---

### 1.2 Top 5 Immediate Actions (P0)

| # | Action | Rule Violated | Effort | Owner |
|---|---|---|---|---|
| 1 | **Remove `Loan_Account` from CRR subject area** — reassign all 141 attributes to `silver.lending` | SA scoping / `sa/subject-areas.md` | S (1 sprint) | Data Architect + Lending SA Steward |
| 2 | **Add `<entity>_sk` (MD5) + `<entity>_bk` to every Silver entity table** | SLV-001, SLV-002 | M (2 sprints) | Silver Pipeline Engineering |
| 3 | **Implement SCD-2 columns** (`effective_from`, `effective_to`, `is_current`, `run_id`, `source_ingestion_date`) on all 7 core entity tables | SLV-003, §2.7 | M (2 sprints) | Silver Pipeline Engineering |
| 4 | **Reclassify or remove the 28 `reference_*` staging tables** — move source copies to Bronze; model canonical Silver entities in 3NF | SLV-006, §2.1 | L (3+ sprints) | Data Architect + Platform Engineering |
| 5 | **Drop `reference_aggregated_pipelines` and `reference_aggregated_individualcontracts`** from Silver — pre-computed aggregations are prohibited | SLV-007, §2.6 | S (1 sprint) | Data Engineer |

---

## 2. Subject Area Architecture Assessment

### 2.1 Scope Verification

**Finding:** `customer_risk_rating` is **not listed as a standalone subject area** in `sa/subject-areas.md`. The canonical taxonomy defines **SA #17 "Risk & Compliance"** (`silver.risk`) which encompasses "KYC Levels, AML Risk Scores, Watchlist Hits, **Customer Risk Ratings**, FATCA/CRS flags, and Basel RWA."

The physical structures use schema `silver_dev.customer_riskrating_scoring`, which diverges in two ways:
1. The environment prefix `silver_dev` indicates a development catalog, not the production `silver` catalog.
2. The schema name `customer_riskrating_scoring` does not align with the canonical schema `silver.risk`.

**Verdict:** The subject area exists as a sub-domain of `silver.risk`. It is permissible to treat it as a bounded sub-area during development, but production deployment must use the `silver.risk` schema and comply with all `silver.risk` governance rules.

**Severity:** 🔴 P0 — Schema name must be aligned before Gold layer references are built.

---

### 2.2 Domain Decomposition

**BIAN v11 Alignment:**  
The Customer Risk Rating domain maps to BIAN Service Domain **"Customer Credit Rating"** (SD-0131) and partially to **"Customer Risk Profile"** (SD-0120). The logical model correctly identifies the primary entities:

| Logical Entity | BIAN Alignment | Assessment |
|---|---|---|
| `Customer_RiskRating & Scoring` | Customer Credit Rating — Rating Assessment | ✅ Correctly scoped |
| `Customer_BureauScore` | Customer Credit Rating — Bureau Score record | ✅ Correctly scoped |
| `Customer_ScoreHistory` | Customer Credit Rating — Rating History | ✅ Correctly scoped |
| `Customer_CollectionSummary` | Collections — Collection Case Summary | ⚠️ Overlap with `silver.lending` Collections sub-area |
| `Customer_DeliquencyHistory` | Customer Credit Rating — Delinquency Record | ⚠️ Overlap with `silver.lending` Collections sub-area |
| `Reference_Risk_Model` | Customer Credit Rating — Risk Model reference | ✅ Correctly scoped |
| `Reference_Bureau_Source` | Customer Credit Rating — Bureau reference | ✅ Correctly scoped |
| `Loan_Account` | Loan — Loan Account | 🔴 **Wrong subject area — must be removed** |

**Issue:** `Customer_CollectionSummary` and `Customer_DeliquencyHistory` overlap with the `silver.lending.collections` sub-area. A governance decision is required: these entities should either live exclusively in `silver.lending` (with CRR subject area referencing via FK only) or be cloned as read-only snapshots in CRR for risk scoring purposes with documented justification.

**Physical Table Proliferation:** 28 out of 39 physical tables (`reference_*`) are source-system table replicas (e.g., `reference_custinfo`, `reference_score`, `reference_addressinfo`). These are Bronze-pattern tables living in Silver, violating domain decomposition and 3NF principles.

---

### 2.3 Identity Strategy

**Finding: CRITICAL GAP**

Per guidelines SLV-001 and SLV-002 (§2.4 of modeling dos/donts):
> "Every Silver entity must carry `<entity>_sk` (MD5 deterministic surrogate) and `<entity>_bk` (business/natural key)."

**Current state across all 7 core entities:**
- ❌ No `<entity>_sk` column defined in any logical entity
- ❌ No `<entity>_bk` column defined in any logical entity
- ❌ Keys are expressed as natural source IDs only (e.g., `Rating id`, `Score id`, `Bureau score id`)
- ❌ Physical tables do not carry SK/BK columns (confirmed by 0-byte and early-created tables)

Without deterministic surrogate keys, Gold layer dimension tables cannot safely join Silver entities across pipeline re-runs, and cross-subject-area FK relationships cannot be established.

---

### 2.4 SCD Strategy

**Finding: CRITICAL GAP**

The standard requires SCD Type 2 as default (SLV-003). Required columns per guideline §2.7:
- `effective_from` TIMESTAMP
- `effective_to` TIMESTAMP  
- `is_current` BOOLEAN
- `run_id` STRING
- `source_ingestion_date` TIMESTAMP

**Current state:**
- The logical model uses `Business Start Date` / `Business End Date` / `Is Active Flag` — non-standard column names that violate NAM-001 (snake_case) and are not the canonical SCD-2 pattern.
- `is_current` is absent.
- `run_id` is absent.
- `source_ingestion_date` is absent.
- The mapping document uses `Create Date` / `Update Date` / `Deleted Date` as audit columns — these are legacy CRM-style columns, not SCD-2 controls.
- `IsLogicallyDeleted` uses PascalCase — violates NAM-001.

The combination of `Business Start Date`/`Business End Date` and `Create Date`/`Update Date` creates confusion about which pair governs SCD-2 validity. Neither pair matches the canonical standard.

---

### 2.5 Cross-Entity Referential Integrity

**Identified FK relationships (from mapping analysis):**

| Child Entity | FK Column | Expected Parent | Physical FK Defined? |
|---|---|---|---|
| `Customer_RiskRating & Scoring` | `CIF (Customer Number)` | `silver.party.party` | ❌ Missing |
| `Customer_RiskRating & Scoring` | `Model id` | `Reference_Risk_Model` | ❌ Missing |
| `Customer_BureauScore` | `CIF (Customer Number)` | `silver.party.party` | ❌ Missing |
| `Customer_BureauScore` | `Bureau id` | `Reference_Bureau_Source` | ❌ Missing |
| `Customer_ScoreHistory` | `CIF (Customer Number)` | `silver.party.party` | ❌ Missing |
| `Customer_ScoreHistory` | `Model id` | `Reference_Risk_Model` | ❌ Missing |
| `Customer_CollectionSummary` | `CIF (Customer Number)` | `silver.party.party` | ❌ Missing |
| `Customer_CollectionSummary` | `Loan account number` | `silver.lending.loan_account` | ❌ Missing |
| `Customer_DeliquencyHistory` | `CIF (Customer Number)` | `silver.party.party` | ❌ Missing |
| `Customer_DeliquencyHistory` | `Loan account number` | `silver.lending.loan_account` | ❌ Missing |

**All FK relationships are undeclared.** This is a P1 gap — without FK enforcement, referential integrity checks (Gate 3 Bronze→Silver) cannot be validated.

---

### 2.6 ETL Mapping Completeness

| Entity | Total Attrs | Mapped (Confirmed) | Not Available | Null/Unknown | % Complete |
|---|---|---|---|---|---|
| `Reference_Risk_Model` | 12 | 6 | 4 | 2 | 50% |
| `Reference_Bureau_Source` | 10 | 7 | 2 | 1 | 70% |
| `Customer_RiskRating & Scoring` | 14 | 9 | 4 | 1 | 64% |
| `Customer_ScoreHistory` | 12 | 8 | 3 | 1 | 67% |
| `Customer_BureauScore` | 13 | 10 | 2 | 1 | 77% |
| `Customer_CollectionSummary` | 16 | 7 | 7 | 2 | 44% |
| `Customer_DeliquencyHistory` | 16 | 7 | 7 | 2 | 44% |
| **Total** | **93** | **54** | **29** | **10** | **58%** |

**Additional issues:**
- All `Attribute_Data_Type` fields in the mapping are **NULL** — no data type specification has been completed for any attribute.
- `Source System Code` / `Source System ID` are marked as "Mapped" but have no source system, table, or column specified (all three source fields are NULL) — these are phantom mappings.
- The Field Classification sheet has no `Attribute_Desc`, `Source of change`, or `Source Systems` populated for any entry.

---

## 3. Entity Inventory

| # | Logical Entity | Physical Table | Schema | Status | Has Data | Priority |
|---|---|---|---|---|---|---|
| 1 | `Reference_Risk_Model` | `reference_risk_model` | `silver_dev.customer_riskrating_scoring` | 🟡 Partial | ✅ Yes (5.8 MB) | P1 |
| 2 | `Reference_Bureau_Source` | `reference_bureau_source` | `silver_dev.customer_riskrating_scoring` | 🔴 Empty | ❌ 0 bytes | P0 |
| 3 | `Customer_RiskRating & Scoring` | `customer_riskrating_and_scoring` | `silver_dev.customer_riskrating_scoring` | 🟡 Partial | ✅ Yes (5 KB) | P0 |
| 3b | *(Duplicate?)* | `customer_riskrating_scoring` | `silver_dev.customer_riskrating_scoring` | ⚠️ Duplicate | ✅ Yes (277 KB) | P0 |
| 4 | `Customer_ScoreHistory` | `customer_score_history` | `silver_dev.customer_riskrating_scoring` | 🟡 Partial | ✅ Yes (13 KB) | P1 |
| 5 | `Customer_BureauScore` | `customer_bureauscore` | `silver_dev.customer_riskrating_scoring` | 🟡 Partial | ✅ Yes (250 KB) | P1 |
| 6 | `Customer_CollectionSummary` | `customer_collection_summary` | `silver_dev.customer_riskrating_scoring` | 🔴 Empty | ❌ 0 bytes | P0 |
| 7 | `Customer_DeliquencyHistory` | `customer_deliquency_history` | `silver_dev.customer_riskrating_scoring` | 🟡 Partial | ✅ Yes (16 KB) | P1 |
| 8 | `Customer_LoanLinkage` | *(none)* | — | 🔴 Unmapped | — | P2 |
| — | `Loan_Account` (141 attrs) | *(none in CRR)* | — | 🔴 Wrong SA | — | P0 (relocate) |
| — | *(No logical entity)* | `reference_custinfo` | `silver_dev.customer_riskrating_scoring` | 🔴 Bronze-in-Silver | ✅ Yes (3.8 MB) | P0 |
| — | *(No logical entity)* | `reference_score` | `silver_dev.customer_riskrating_scoring` | 🔴 Bronze-in-Silver | ✅ Yes (229 KB) | P0 |
| — | *(No logical entity)* | `reference_history48months` | `silver_dev.customer_riskrating_scoring` | 🔴 Bronze-in-Silver | ✅ Yes (1.2 MB) | P0 |
| — | *(No logical entity)* | `reference_aggregated_pipelines` | `silver_dev.customer_riskrating_scoring` | 🔴 Aggregation/Prohibited | ✅ Yes (3.7 MB) | P0 |
| — | *(No logical entity)* | `reference_aggregated_individualcontracts` | `silver_dev.customer_riskrating_scoring` | 🔴 Aggregation/Prohibited | ✅ Yes (12 MB) | P0 |
| — | *(No logical entity)* | `reference_financialsummary` | `silver_dev.customer_riskrating_scoring` | ⚠️ Investigate | ✅ Yes (41 KB) | P1 |
| — | *(+22 other reference_* staging tables)* | Various | `silver_dev.customer_riskrating_scoring` | 🔴 Bronze-in-Silver | Mixed | P0 |

**Total physical tables:** 39 (including 1 null/blank row)  
**Tables with logical entity mapping:** 8 (core entities)  
**Bronze-like staging replicas with no logical model:** 28  
**Empty tables (0 bytes):** 4 (`reference_bureau_source`, `customer_collection_summary`, `reference_cases`, `reference_channel_type`)

---

## 4. Per-Entity Assessments

---

### 4.1 Entity: `Reference_Risk_Model`

**Physical Table:** `silver_dev.customer_riskrating_scoring.reference_risk_model`  
**Data Size:** 6.08 MB | 1,936 files  
**Confidence:** 0.90

#### 4.1a Industry Fitness

| Dimension | Finding |
|---|---|
| **Entity Boundary** | Acceptable — a risk model reference entity describing scoring model metadata is standard for BIAN Customer Credit Rating SD |
| **Grain** | One row per risk model version; correct grain |
| **Scope** | ✅ Well-scoped; only 12 attributes, all model-metadata |
| **BIAN Alignment** | Aligns with BIAN "Risk Model" component in Customer Credit Rating (SD-0131) |

**Verdict:** Industry-fit entity with correct scope. Structural and naming issues present.

#### 4.1b Attribute-Level Review

| Attribute (Logical) | Physical Column | Type (Logical) | Expected Type | Mapped? | Key Type | SCD | Naming | Issues |
|---|---|---|---|---|---|---|---|---|
| Model id | *(assumed `model_id`)* | STRING | STRING NOT NULL (PK) | Partial | PK — but no `model_sk`/`model_bk` | N/A | ⚠️ Missing `model_sk`, `model_bk` | SLV-001, SLV-002 |
| Model name | `model_name` | STRING | STRING | Partial | — | SCD-2 tracked | ✅ | — |
| Model type | `model_type` | STRING | STRING → should map to code | Partial | — | SCD-2 tracked | ✅ | Should use canonical `model_type_code` |
| Applicable segment code | `applicable_segment_code` | STRING | STRING | Mapped (Bureauone.InquiryResponseHeader.CustomerCode) | — | SCD-2 | ✅ | Source col `CustomerCode` ≠ `applicable_segment_code` — transformation undocumented |
| Valid from | `valid_from` | STRING | TIMESTAMP | Mapped (Bureauone.CustInfo.CreatedOn) | — | SCD effective start | ⚠️ | Should be `valid_from_timestamp` TIMESTAMP; mapping to `CreatedOn` is semantically questionable for a model validity date |
| Valid to | `valid_to` | STRING | TIMESTAMP | Mapped (Bureauone.CustInfo.UpdatedOn) | — | SCD effective end | ⚠️ | Mapping model validity end to `UpdatedOn` is incorrect — model validity is not the same as record update time |
| Source system code | *(assumed)* | STRING | STRING | Mapped (no source specified) | Metadata | — | ✅ | Phantom mapping — no source system/table/column specified in mapping doc |
| Source system id | *(assumed)* | STRING | STRING | Mapped (no source specified) | Metadata | — | ✅ | Phantom mapping |
| IsLogicallyDeleted | *(assumed)* | STRING | BOOLEAN | Not Available | Metadata | — | 🔴 | Violates NAM-001 (PascalCase); should be `is_deleted` BOOLEAN |
| Create date | `create_date` | STRING | TIMESTAMP NOT NULL | Mapped (no source) | Metadata | — | ✅ | Type must be TIMESTAMP not STRING |
| Update date | `update_date` | STRING | TIMESTAMP | Mapped (no source) | Metadata | — | ✅ | Type must be TIMESTAMP |
| Deleted date | `deleted_date` | STRING | TIMESTAMP | Not Available | Metadata | — | ✅ | Type must be TIMESTAMP |
| *(Missing)* | — | — | TIMESTAMP NOT NULL | — | `effective_from` | SCD-2 | 🔴 | Required SCD-2 column absent |
| *(Missing)* | — | — | TIMESTAMP | — | `effective_to` | SCD-2 | 🔴 | Required SCD-2 column absent |
| *(Missing)* | — | — | BOOLEAN NOT NULL | — | `is_current` | SCD-2 | 🔴 | Required SCD-2 column absent |
| *(Missing)* | — | — | STRING | — | `run_id` | Metadata | 🔴 | Required audit column absent |
| *(Missing)* | — | — | TIMESTAMP | — | `source_ingestion_date` | Metadata | 🔴 | Required audit column absent |
| *(Missing)* | — | — | STRING NOT NULL | — | `model_sk` | SK | 🔴 | SLV-001 violation |
| *(Missing)* | — | — | STRING NOT NULL | — | `model_bk` | BK | 🔴 | SLV-002 violation |

**Key Finding:** `valid_to` is mapped from `Bureauone.CustInfo.UpdatedOn` — this conflates "when the source record was updated" with "when the risk model's validity period ends." These are semantically different dates. Model validity end date should come from the risk model governance system, not a customer info update timestamp.

#### 4.1c Metadata Completeness

| Required Column | Present? | Correct Type? |
|---|---|---|
| `source_system_code` | ✅ Defined in logical | ⚠️ No source specified in mapping |
| `source_system_id` | ✅ Defined in logical | ⚠️ No source specified in mapping |
| `create_date` | ✅ | ⚠️ STRING not TIMESTAMP |
| `update_date` | ✅ | ⚠️ STRING not TIMESTAMP |
| `delete_date` | ✅ (`deleted_date`) | ⚠️ STRING not TIMESTAMP |
| `is_active_flag` | ⚠️ `IsLogicallyDeleted` only | 🔴 Non-standard name |
| `effective_from` | 🔴 Absent | — |
| `effective_to` | 🔴 Absent | — |
| `is_current` | 🔴 Absent | — |
| `run_id` | 🔴 Absent | — |
| `source_ingestion_date` | 🔴 Absent | — |
| `record_source` / DQ columns | 🔴 Absent | — |

**Score: 3/11 required metadata columns present and correctly typed. Critical.**

---

### 4.2 Entity: `Reference_Bureau_Source`

**Physical Table:** `silver_dev.customer_riskrating_scoring.reference_bureau_source`  
**Data Size:** 0 bytes | 0 files (EMPTY)  
**Confidence:** 0.88

#### 4.2a Industry Fitness

| Dimension | Finding |
|---|---|
| **Entity Boundary** | Acceptable — a bureau reference entity capturing credit bureau metadata (Bureauone, AECB, etc.) is standard in UAE retail banking |
| **Grain** | One row per credit bureau; correct grain |
| **BIAN Alignment** | Aligns with BIAN "External Reference" component within Customer Credit Rating |

**Critical Operational Gap:** The table is **empty** (0 bytes). `customer_bureauscore` has a FK dependency on `bureau_id`, meaning all bureau score records currently have an unresolvable foreign key. This is a Gate 3 referential integrity failure.

#### 4.2b Attribute-Level Review

| Attribute | Mapped? | Source | Issues |
|---|---|---|---|
| Bureau id | Mapped | Bureauone.SCORE.Bureauid | PK — but no `bureau_source_sk` / `bureau_source_bk` |
| Bureau name | Mapped | Bureauone.SCORE.Name | ✅ |
| Country code | Mapped | Bureauone.AddressInfo.Country_Code | ⚠️ Should resolve to `silver.reference.country_code` canonical value |
| Report format | Mapped | Bureauone.SCORE.Value | 🔴 Mapping to `Value` column is semantically incorrect — SCORE.Value is a numeric score value, not a report format |
| Source system code | Phantom mapping | None | ⚠️ |
| Source system id | Phantom mapping | None | ⚠️ |
| IsLogicallyDeleted | Mapped | None | 🔴 NAM-001 violation |
| Create date | Mapped | None | ⚠️ STRING not TIMESTAMP |
| Update date | Mapped | None | ⚠️ STRING not TIMESTAMP |
| Deleted date | Mapped | None | ⚠️ STRING not TIMESTAMP |
| `bureau_source_sk` | ❌ Missing | — | SLV-001 |
| `bureau_source_bk` | ❌ Missing | — | SLV-002 |
| `effective_from` | ❌ Missing | — | SLV-003 |
| `effective_to` | ❌ Missing | — | SLV-003 |
| `is_current` | ❌ Missing | — | SLV-003 |

**Semantic Error (P0):** `Report format` is mapped to `Bureauone.SCORE.Value` which is a numeric score value column. This mapping is demonstrably wrong and must be corrected.

#### 4.2c Metadata Completeness

**Score: 3/11 required metadata columns present. Empty table. P0.**

---

### 4.3 Entity: `Customer_RiskRating & Scoring`

**Physical Tables:** `customer_riskrating_and_scoring` (5 KB) AND `customer_riskrating_scoring` (277 KB)  
**Confidence:** 0.93

#### 4.3a Industry Fitness

| Dimension | Finding |
|---|---|
| **Entity Boundary** | ⚠️ Concern — two physical tables exist for the same logical entity. `customer_riskrating_and_scoring` and `customer_riskrating_scoring` appear to be different iterations of the same entity. This creates ambiguity about which is authoritative. |
| **Grain** | One row per customer-per-rating-event-per-model; correct |
| **Scope** | ✅ Well-scoped core entity for the subject area |
| **BIAN Alignment** | Directly maps to BIAN Customer Credit Rating — "Rating Assessment" |

**Duplicate Table Issue (P0):** The existence of both `customer_riskrating_and_scoring` AND `customer_riskrating_scoring` in the same schema is a critical architecture defect. Both appear to serve the same purpose. The larger file (`customer_riskrating_scoring`, 277 KB vs 5 KB) is likely the active table, while the smaller one is a leftover from an earlier naming iteration. A formal decommissioning decision is required.

#### 4.3b Attribute-Level Review

| Attribute (Logical) | Mapped? | Source | Issues |
|---|---|---|---|
| Rating id | Mapped | Bureauone.SCORE.Reference_Number | PK — no `customer_risk_rating_sk` / `customer_risk_rating_bk`; SLV-001/002 |
| CIF (Customer Number) | Mapped | Bureauone.CustInfo.CustomerId | FK to `silver.party` — not declared; `customer_bk` should be the FK column name |
| Model id | Not Available | — | 🔴 FK to `Reference_Risk_Model` — not available = referential integrity broken |
| Rating value | Mapped | Bureauone.SCORE.Value | ⚠️ `Value` is generic; semantics unverified. Type should be DECIMAL not STRING |
| Rating score | Mapped | Bureauone.SCORE.CreditRating | ⚠️ May be a code (e.g., A/B/C) — if so, must resolve via `silver.reference.code_mapping` (SLV-004) |
| Rating date | Mapped | Bureauone.SCORE.Createdon | ⚠️ Type STRING — must be DATE or TIMESTAMP |
| Rating valid till | Mapped | Bureauone.CustInfo.UpdatedOn | 🔴 **Semantic error** — `UpdatedOn` is the customer record last-update time, not the rating validity expiry. Rating validity expiry requires a dedicated source field. |
| Source | Not Available | — | ⚠️ Should reference `reference_bureau_source.bureau_id` |
| Source system code | Phantom mapping | — | ⚠️ |
| Source system id | Phantom mapping | — | ⚠️ |
| IsLogicallyDeleted | Not Available | — | 🔴 NAM-001 |
| Create date | Mapped | — | ⚠️ STRING → TIMESTAMP |
| Update date | Mapped | — | ⚠️ STRING → TIMESTAMP |
| Deleted date | Not Available | — | ⚠️ STRING → TIMESTAMP |
| Business Start Date | Mapped | — | 🔴 Non-standard SCD column name; must become `effective_from` TIMESTAMP |
| Business End Date | Mapped | — | 🔴 Non-standard SCD column name; must become `effective_to` TIMESTAMP |
| Is Active Flag | Mapped | — | ⚠️ Non-standard; must become `is_current` BOOLEAN |
| `customer_risk_rating_sk` | ❌ Missing | — | SLV-001 |
| `customer_risk_rating_bk` | ❌ Missing | — | SLV-002 |
| `run_id` | ❌ Missing | — | §2.7 |
| `source_ingestion_date` | ❌ Missing | — | §2.7 |

**Critical Semantic Error:** `Rating Valid Till` is mapped to `Bureauone.CustInfo.UpdatedOn`. "Rating Valid Till" is a business-critical attribute for credit risk governance (BCBS 239) representing the expiry date of the risk assessment. Mapping it to a record update timestamp is factually incorrect and constitutes a data quality failure.

#### 4.3c Metadata Completeness

**Score: 4/11 (with non-standard names). P0.**

---

### 4.4 Entity: `Customer_ScoreHistory`

**Physical Table:** `silver_dev.customer_riskrating_scoring.customer_score_history`  
**Data Size:** 13 KB | 2 files  
**Confidence:** 0.89

#### 4.4a Industry Fitness

| Dimension | Finding |
|---|---|
| **Entity Boundary** | ✅ Correct — a score history entity tracking customer scores over time is essential for trend analysis and IFRS 9 staging |
| **Grain** | One row per customer-per-score-event; correct |
| **Scope** | ✅ Well-scoped; 12 attributes |
| **BIAN Alignment** | Maps to BIAN Customer Credit Rating — "Rating History" |

**Note:** `Customer_ScoreHistory` and `Customer_RiskRating & Scoring` may have overlapping purpose. The distinction is that ScoreHistory tracks bureau/model scores (quantitative), while RiskRating tracks the assigned rating band (qualitative). This distinction must be documented.

#### 4.4b Attribute-Level Review

| Attribute | Mapped? | Source | Issues |
|---|---|---|---|
| Score id | Mapped | Bureauone.Score | PK — no `customer_score_history_sk` / `_bk`; SLV-001/002 |
| CIF (Customer Number) | Mapped | Bureauone.CustInfo.CustomerId | FK undeclared |
| Model id | Not Available | — | 🔴 FK to `Reference_Risk_Model` broken |
| Score value | Mapped | Bureauone.Score.Value | ⚠️ Type must be DECIMAL(10,4) not STRING |
| Score date | Mapped | Bureauone.Score.ScoreDate | ⚠️ Type must be DATE not STRING |
| Source | Mapped | Bureauone.Score.BureauId | ✅ |
| Source system code | Phantom | — | ⚠️ |
| Source system id | Phantom | — | ⚠️ |
| IsLogicallyDeleted | Not Available | — | 🔴 NAM-001 |
| Create date | Mapped | — | ⚠️ STRING → TIMESTAMP |
| Update date | Mapped | — | ⚠️ STRING → TIMESTAMP |
| Deleted date | Not Available | — | ⚠️ STRING → TIMESTAMP |
| `customer_score_history_sk` | ❌ Missing | — | SLV-001 |
| `customer_score_history_bk` | ❌ Missing | — | SLV-002 |
| `effective_from` | ❌ Missing | — | SLV-003 |
| `effective_to` | ❌ Missing | — | SLV-003 |
| `is_current` | ❌ Missing | — | SLV-003 |
| `run_id` | ❌ Missing | — | §2.7 |
| `source_ingestion_date` | ❌ Missing | — | §2.7 |

#### 4.4c Metadata Completeness

**Score: 3/11. P1.**

---

### 4.5 Entity: `Customer_BureauScore`

**Physical Table:** `silver_dev.customer_riskrating_scoring.customer_bureauscore`  
**Data Size:** 250 KB | 5 files  
**Confidence:** 0.91

#### 4.5a Industry Fitness

| Dimension | Finding |
|---|---|
| **Entity Boundary** | ✅ Correct — captures individual bureau-sourced credit scores per customer |
| **Grain** | One row per customer-per-bureau-per-score-date; correct |
| **BIAN Alignment** | Maps to BIAN Customer Credit Rating — external bureau score record |
| **UAE Context** | AECB (Al Etihad Credit Bureau) is the mandatory UAE credit bureau; scores from AECB must be captured. The `AECB_Range` attribute (extra attribute in mapping not in logical model) confirms AECB-specific data is being handled. |

#### 4.5b Attribute-Level Review

| Attribute | Mapped? | Source | Issues |
|---|---|---|---|
| Bureau score id | Mapped | Bureauone.CUSTINFO.Reference_Number | PK — no `customer_bureau_score_sk` / `_bk`; SLV-001/002 |
| CIF (Customer Number) | Mapped (Transformation) | Bureauone.CUSTINFO.CUSTOMERID | ✅ Transformation noted; transformation logic undocumented |
| Bureau id | Mapped | Bureauone.SCORE.Bureauid | FK to `reference_bureau_source` — undeclared and table is empty |
| Score value | Mapped | Bureauone.SCORE.Value | ⚠️ Must be DECIMAL not STRING |
| Score grade | Mapped | Bureauone.lkp_ScoreRange.ItemDesc | ⚠️ Grade is a code — should map via `silver.reference.code_mapping` (SLV-004); direct lookup from source LKP table acceptable only if mapped to canonical code |
| Score date | Mapped | Bureauone.SCORE.Createdon | ⚠️ Must be DATE not STRING |
| Report reference | Mapped | Bureauone.SCORE.Description | ✅ |
| Source system code | Phantom | — | ⚠️ |
| Source system id | Phantom | — | ⚠️ |
| IsLogicallyDeleted | Not Available | — | 🔴 NAM-001 |
| Create date | Mapped | — | ⚠️ STRING → TIMESTAMP |
| Update date | Mapped | — | ⚠️ STRING → TIMESTAMP |
| Deleted date | Not Available | — | — |
| AECB_Range | None (no status) | Bureauone.SCORE.RANGE | 🔴 **Attribute exists in mapping but NOT in logical model** — undocumented addition; must be added to logical model or removed |
| `customer_bureau_score_sk` | ❌ Missing | — | SLV-001 |
| `customer_bureau_score_bk` | ❌ Missing | — | SLV-002 |
| `effective_from` | ❌ Missing | — | SLV-003 |
| `effective_to` | ❌ Missing | — | SLV-003 |
| `is_current` | ❌ Missing | — | SLV-003 |
| `run_id` | ❌ Missing | — | §2.7 |
| `source_ingestion_date` | ❌ Missing | — | §2.7 |

**Model-Mapping Divergence:** `AECB_Range` appears in the mapping document but not in the logical model. This is a governance gap — all physical attributes must have a corresponding logical model definition. The attribute must be formally added to the Erwin model or removed from the physical table.

#### 4.5c Metadata Completeness

**Score: 3/11. PII concern: `CIF (Customer Number)` identifies individual customers — must be masked per PII controls (SHA-256) unless in restricted container.**

---

### 4.6 Entity: `Customer_CollectionSummary`

**Physical Table:** `silver_dev.customer_riskrating_scoring.customer_collection_summary`  
**Data Size:** 0 bytes | 0 files (EMPTY)  
**Confidence:** 0.86

#### 4.6a Industry Fitness

| Dimension | Finding |
|---|---|
| **Entity Boundary** | ⚠️ Scope conflict — collection summary data also exists in `silver.lending.collections` sub-area (per `sa/subject-areas.md`). Presence here creates potential duplication. |
| **Grain** | One row per customer-per-collection-case; acceptable for risk scoring purpose |
| **BIAN Alignment** | Overlaps BIAN Collections SD and Customer Credit Rating SD |
| **Justification Required** | A CRR subject area may legitimately hold a **snapshot** of collection status for risk scoring purposes, but this must be documented as a read-only denormalized copy. Without documentation, this is a scoping violation. |

**Critical:** The table is empty. No pipeline is writing data. This entity is planned but not operational.

#### 4.6b Attribute-Level Review

| Attribute | Mapped? | Status | Issues |
|---|---|---|---|
| Collection id | Mapped | Bureauone.LinkInfo.CollectionId | PK — no SK/BK |
| CIF (Customer Number) | Mapped | Bureauone.CustInfo.CustomerId | FK undeclared |
| Loan Facility ID | Mapped | Bureauone.CustInfo.Product_Code | 🔴 **Semantic error** — `Product_Code` is a product identifier, not a loan facility ID |
| Loan Account Number | Mapped | Bureauone.CustInfo.AccountNumber | ✅ |
| Case Open Date | Not Available | — | 🔴 Missing — critical for delinquency tracking |
| Current Bucket | Not Available | — | 🔴 Missing — bucket (1-30 DPD, 31-60, etc.) is a key risk metric |
| Status | Mapped | Bureauone.custinfo.Status | ⚠️ Raw source code — must resolve via `silver.reference.code_mapping` (SLV-004) |
| Last Activity Date | Not Available | — | 🔴 Missing |
| Source system code | Phantom | — | ⚠️ |
| Source system id | Phantom | — | ⚠️ |
| IsLogicallyDeleted | Not Available | — | 🔴 NAM-001 |
| Create date | Not Available | — | 🔴 Missing |
| Update date | Not Available | — | 🔴 Missing |
| Deleted date | Not Available | — | 🔴 Missing |
| Account Number | Mapped | Bureauone.AccountDetails.AccountNumber | ⚠️ Duplicate of `Loan Account Number`? Relationship unclear |
| Account Type | Mapped | Bureauone.AccountDetails.AccountType | ⚠️ Raw source code — must use canonical mapping |
| `customer_collection_summary_sk` | ❌ Missing | — | SLV-001 |
| `customer_collection_summary_bk` | ❌ Missing | — | SLV-002 |

**Semantic Error:** `Loan Facility ID` is mapped to `Bureauone.CustInfo.Product_Code`. A product code identifies a product type (e.g., "AUTO LOAN"), not a specific loan facility instance. The actual loan facility ID (a unique account/agreement identifier) must come from a different source column.

#### 4.6c Metadata Completeness

**Score: 1/11 (source_system_code only as phantom). Empty table. P0.**

---

### 4.7 Entity: `Customer_DeliquencyHistory`

**Physical Table:** `silver_dev.customer_riskrating_scoring.customer_deliquency_history`  
**Data Size:** 16 KB | 4 files  
**Confidence:** 0.90

> **Note on entity name:** The entity is misspelled as `Customer_Delinquency_History` throughout the logical model, mapping document, and physical table — "Deliquency" instead of "Delinquency". This violates NAM-001 and must be corrected before Gold layer references proliferate.

#### 4.7a Industry Fitness

| Dimension | Finding |
|---|---|
| **Entity Boundary** | ✅ Correct conceptually — DPD (Days Past Due) history is a key credit risk input for IFRS 9 staging and BCBS 239 |
| **Grain** | One row per customer-per-account-per-DPD-date; correct |
| **BIAN Alignment** | BIAN Customer Credit Rating — "Delinquency Record" |
| **IFRS 9 Relevance** | DPD history is the primary trigger for IFRS 9 Stage 2/3 classification. This entity is CDE-critical. |
| **Scoping Concern** | Same overlap with `silver.lending.collections` as `Customer_CollectionSummary`. Governance decision required. |

#### 4.7b Attribute-Level Review

| Attribute | Mapped? | Source | Issues |
|---|---|---|---|
| DPD id | Not Available | — | 🔴 PK not mapped; no `customer_delinquency_history_sk` / `_bk` |
| CIF (Customer Number) | Mapped | Bureauone.CustInfo.CustomerId | FK undeclared |
| Loan Facility ID | Mapped | Bureauone.CustInfo.Product_Code | 🔴 Same semantic error as 4.6 — `Product_Code` ≠ Loan Facility ID |
| Loan Account Number | Mapped | Bureauone.CustInfo.AccountNumber | ✅ |
| DPD Date | Mapped | Bureauone.History48Months.PaymentDueDate | ⚠️ `PaymentDueDate` is the date payment was due — this may not be the date the DPD event occurred. Needs clarification. |
| Days Past Due (DPD) | Mapped | Bureauone.History48Months.Days_Past_Due | ✅ Good mapping |
| Delinquency Bucket | Not Available | — | 🔴 Missing — critical IFRS 9 attribute |
| Resolved Flag | Not Available | — | 🔴 Missing — cannot determine if DPD was cured |
| Source system code | Phantom | — | ⚠️ |
| Source system id | Phantom | — | ⚠️ |
| IsLogicallyDeleted | Not Available | — | 🔴 NAM-001 |
| Create date | Not Available | — | 🔴 |
| Update date | Not Available | — | 🔴 |
| Deleted date | Not Available | — | 🔴 |
| Account Number | Mapped | Bureauone.AccountDetails.AccountNumber | ⚠️ Duplicate of `Loan Account Number` — relationship unclear |
| Account Type | Mapped | Bureauone.AccountDetails.AccountType | ⚠️ Raw code — needs canonical mapping |
| Business Start Date | — | — | 🔴 Non-standard SCD column |
| Business End Date | — | — | 🔴 Non-standard SCD column |
| Is Active Flag | — | — | 🔴 Non-standard |
| `customer_delinquency_history_sk` | ❌ Missing | — | SLV-001 |
| `customer_delinquency_history_bk` | ❌ Missing | — | SLV-002 |
| `effective_from` | ❌ Missing | — | SLV-003 |
| `effective_to` | ❌ Missing | — | SLV-003 |
| `is_current` | ❌ Missing | — | SLV-003 |
| `run_id` | ❌ Missing | — | §2.7 |
| `source_ingestion_date` | ❌ Missing | — | §2.7 |

**IFRS 9 / BCBS 239 Critical Gaps:**
- `Delinquency Bucket` (Not Available) — Stage 2 trigger; its absence is a regulatory reporting deficiency.
- `Resolved Flag` (Not Available) — required to compute cure rates and re-stage from Stage 2 to Stage 1.
- Entity name misspelling (`Deliquency`) must be corrected.

#### 4.7c Metadata Completeness

**Score: 2/11. IFRS 9 critical entity with major gaps. P0.**

---

### 4.8 Entity: `Customer_LoanLinkage`

**Physical Table:** None  
**Confidence:** 0.75

#### 4.8a Assessment

The `Customer_LoanLinkage` entity appears in the logical model with 1 attribute (null value — essentially an empty entity shell). There is no physical table, no mapping, and no attribute definition.

**Verdict:** This entity is a placeholder that was never developed. It likely represents the association between a customer and their loan accounts for risk scoring purposes. This linkage is already implied by FK relationships on `Customer_CollectionSummary` and `Customer_DeliquencyHistory`. Clarification is needed: is this a bridge/associative table, or is it redundant?

**Recommendation:** Define the entity's purpose, grain, and attributes. If it serves as a bridge entity between customers and loan accounts for risk context, it should include `customer_bk`, `loan_account_bk`, `link_type_code`, `effective_from`, `effective_to`, `is_current`. If redundant, deprecate.

**Priority:** P2 (backlog — no data impact until entity is formally defined).

---

### 4.9 Non-Logical Physical Tables (Bronze-in-Silver Pattern)

The following 28 physical tables have **no corresponding logical entity** in the approved `Customer Risk Rating and Score` logical model. They are direct replicas of source tables from the Bureauone credit bureau system and violate SLV-006 (3NF), §2.1 (3NF), and §2.2 (subject-area isolation).

| Table | Purpose Inferred | Size | Recommendation |
|---|---|---|---|
| `reference_custinfo` | Customer info from Bureauone | 3.8 MB | Move to Bronze; derive Silver entity |
| `reference_score` | Score records from Bureauone | 229 KB | Move to Bronze; already mapped to core entities |
| `reference_history48months` | 48-month payment history | 1.2 MB | Move to Bronze; derives `customer_deliquency_history` |
| `reference_addressinfo` | Address data from Bureauone | 252 KB | Move to Bronze; address belongs to `silver.party` |
| `reference_identityinfo` | Identity documents | 991 KB | Move to Bronze; belongs to `silver.party` |
| `reference_telephoneinfo` | Phone numbers | 408 KB | Move to Bronze; belongs to `silver.party` |
| `reference_personalinfo` | Personal info | 342 KB | Move to Bronze; belongs to `silver.party` |
| `reference_employmentinformation` | Employment info | 169 KB | Move to Bronze; belongs to `silver.party` |
| `reference_inquiryresponseheader` | Bureau inquiry headers | 135 KB | Move to Bronze; possibly source for `reference_risk_model` |
| `reference_inquiryhistory` | Bureau inquiry history | 64 KB | Move to Bronze |
| `reference_contractssummary` | Contract summary | 42 KB | Move to Bronze; overlap with `silver.contract` |
| `reference_contractslink` | Contract links | 17 KB | Move to Bronze |
| `reference_contractsdistribution` | Contract distribution | 49 KB | Move to Bronze |
| **`reference_aggregated_pipelines`** | **Pre-aggregated pipeline data** | **3.7 MB** | **🔴 DELETE — aggregations prohibited in Silver** |
| **`reference_aggregated_individualcontracts`** | **Pre-aggregated contracts** | **12 MB** | **🔴 DELETE — aggregations prohibited in Silver** |
| `reference_financialsummary` | Financial summary | 42 KB | 🔴 Investigate — "summary" implies aggregation |
| `reference_bulkportfolio_characteristics` | Portfolio characteristics | 117 KB | 🔴 Investigate — bulk/portfolio implies aggregation |
| `reference_accountdetails` | Account details | 136 KB | Move to Bronze; belongs to `silver.account` |
| `reference_salarycredits` | Salary credits | 22 KB | Move to Bronze; belongs to `silver.transaction` or `silver.party` |
| `reference_salarycredits_history` | Salary credits history | 153 KB | Move to Bronze |
| `reference_paymentorder` | Payment orders | 25 KB | Move to Bronze; belongs to `silver.payment` |
| `reference_nameenhistory` | Name history | 28 KB | Move to Bronze; belongs to `silver.party` |
| `reference_dobhistory` | Date of birth history | 28 KB | Move to Bronze; belongs to `silver.party` |
| `reference_residentflaghistory` | Residency flag history | 17 KB | Move to Bronze; belongs to `silver.party` |
| `reference_titlehistory` | Title history | 17 KB | Move to Bronze; belongs to `silver.party` |
| `reference_custinfo_company` | Corporate customer info | 65 KB | Move to Bronze; belongs to `silver.party` |
| `reference_lkp_inquiry_purposes` | Lookup: inquiry purposes | 21 KB | Move to `silver.reference` if canonical lookup |
| `reference_lkp_contractstatustype` | Lookup: contract status types | 14 KB | Move to `silver.reference` |
| `reference_lkp_scorerange` | Lookup: score ranges | 14 KB | Move to `silver.reference` |
| `reference_cases` | Collection cases | 0 bytes | 🔴 Empty; belongs to `silver.lending.collections` if used |
| `reference_channel_type` | Channel types | 0 bytes | Move to `silver.reference` |

**Structural Pattern Violation:** These tables replicate raw source data inside Silver. Their naming (`reference_*`) is misleading — they are not Silver reference data entities but Bronze staging copies. The Silver schema must be restructured so only canonicalized, 3NF-normalized entities remain.

---

## 5. Denormalization Register

Per SLV-006: Silver must be in Third Normal Form. Denormalization is only acceptable at Gold layer.

| # | Tables Involved | Denormalized Attribute(s) | Classification | Justification Required | Verdict |
|---|---|---|---|---|---|
| 1 | `customer_deliquency_history` includes `account_number` + `account_type` | Account attributes duplicated in DPD history | **Unnecessary** | ❌ None provided | 🔴 Remove; resolve via FK to `silver.account` |
| 2 | `customer_collection_summary` includes `account_number` + `account_type` | Same account attributes duplicated | **Unnecessary** | ❌ None provided | 🔴 Remove; resolve via FK |
| 3 | `customer_riskrating_and_scoring` has both `rating_value` (numeric) + `rating_score` (grade) | Score and grade co-located | **Acceptable** | Score + band are logically related | ✅ Document justification |
| 4 | `reference_*` tables: `reference_custinfo`, `reference_personalinfo`, `reference_addressinfo`, `reference_telephoneinfo`, `reference_identityinfo` — all party attributes co-located in CRR schema | Party identity attributes in CRR | **Unnecessary** | ❌ None provided | 🔴 Remove; belongs to `silver.party` |
| 5 | `reference_aggregated_pipelines`, `reference_aggregated_individualcontracts` | Aggregated/computed metrics in Silver | **Prohibited** | N/A | 🔴 Delete immediately (SLV-007) |
| 6 | `loan_facility_id` in both `customer_collection_summary` and `customer_deliquency_history` | Loan account FK repeated across two entities | **Acceptable** | Each entity has independent grain | ✅ Acceptable if FK properly declared to `silver.lending` |
| 7 | `reference_financialsummary` — financial summary data in CRR schema | Financial data belongs in `silver.gl` or `silver.transaction` | **Unnecessary** | ❌ None provided | ⚠️ Investigate and relocate |

---

## 6. Guideline Compliance Summary

### 6.1 SLV Rules

| Rule | Description | Status | Finding |
|---|---|---|---|
| **SLV-001** | `<entity>_sk` MD5 surrogate on every entity | 🔴 FAIL | Zero entities carry `_sk` column; identity strategy entirely absent |
| **SLV-002** | `<entity>_bk` business key on every entity | 🔴 FAIL | Zero entities carry `_bk` column; natural source IDs used directly |
| **SLV-003** | SCD-2 default (`effective_from`, `effective_to`, `is_current`) | 🔴 FAIL | SCD-2 columns absent on all entities; non-standard `Business Start Date`/`Business End Date` used |
| **SLV-004** | No raw source-system codes; use `silver.reference.code_mapping` | 🔴 FAIL | `Status`, `Account Type`, `Score Grade` (and others) store raw source codes without canonical mapping |
| **SLV-005** | DQ-gated writes; quarantine table | 🔴 FAIL | No quarantine table exists; no `data_quality_status` / `dq_issues` columns on any entity |
| **SLV-006** | 3NF; no derived values/aggregations | 🔴 FAIL | 28 Bronze-replica tables in Silver; multiple transitive dependencies in collection/delinquency entities |
| **SLV-007** | No pre-computed metrics in Silver | 🔴 FAIL | `reference_aggregated_pipelines` (3.7 MB) and `reference_aggregated_individualcontracts` (12 MB) are pre-computed aggregations |
| **SLV-008** | All mandatory metadata columns present | 🔴 FAIL | `effective_from`, `effective_to`, `is_current`, `run_id`, `source_ingestion_date` missing; DQ columns missing |
| **SLV-009** | All timestamps UTC | ⚠️ UNKNOWN | All types are STRING in logical model — cannot verify UTC compliance; must be enforced in DDL |
| **SLV-010** | Monetary amounts in fils (smallest AED unit) | N/A | No monetary amount columns in CRR entities |

### 6.2 NAM Rules

| Rule | Description | Status | Finding |
|---|---|---|---|
| **NAM-001** | snake_case, lowercase identifiers | 🔴 FAIL | `IsLogicallyDeleted` (PascalCase), `Business Start Date` (spaces + PascalCase), `Is Active Flag` (spaces), `CIF(Customer Number)` violate convention |
| **NAM-002** | Catalog/schema: `silver.<subject_area>` | 🔴 FAIL | Physical schema is `silver_dev.customer_riskrating_scoring` — wrong catalog and wrong schema name |
| **NAM-003** | Table naming: `<entity>` for Silver entities | 🟡 PARTIAL | Core entity tables (`customer_riskrating_and_scoring`, `customer_bureauscore`) follow pattern; however `reference_*` tables use a non-standard naming pattern for what should be source-staging tables |
| **NAM-005** | No SQL reserved words as identifiers | ✅ PASS | No reserved words detected in entity/attribute names |

### 6.3 Additional Modeling Rules (§11)

| Rule | Status | Finding |
|---|---|---|
| **§2.1 — 3NF** | 🔴 FAIL | 28 non-normalized tables; repeating account attributes in collection/delinquency entities |
| **§2.3 — SCD-2** | 🔴 FAIL | Non-standard date columns used; `is_current` absent |
| **§2.4 — Deterministic SK** | 🔴 FAIL | No surrogate keys defined anywhere |
| **§2.5 — Code Canonicalization** | 🔴 FAIL | Raw codes stored; no `silver.reference.code_mapping` joins evidenced |
| **§2.6 — No Aggregations** | 🔴 FAIL | Two aggregated tables present |
| **§2.7 — Mandatory Audit Columns** | 🔴 FAIL | 5 of 11 mandatory columns absent |
| **§2.11 — Erwin Model Before DDL** | 🟡 PARTIAL | Logical model exists but `Loan_Account` is in wrong SA; `Customer_LoanLinkage` is a shell; `AECB_Range` attribute not in logical model |
| **§2.12 — Data Catalog Registration** | 🔴 FAIL | All 39 table descriptions are `null` in physical structures metadata |
| **Gate 2 — Error Rate Threshold** | 🔴 UNVERIFIED | No DQ pipeline code or quarantine tables evidence; cannot verify 5% threshold enforcement |
| **Gate 3 — Referential Integrity** | 🔴 FAIL | `reference_bureau_source` is empty; all `bureau_id` FK references in `customer_bureauscore` are unresolvable |
| **PII Controls** | ⚠️ RISK | `CIF (Customer Number)` is a customer identifier — requires SHA-256 masking or restricted container |

---

## 7. Remediation Plan

### 7.1 Prioritized Action List

#### P0 — Immediate (Before Any Production Deployment)

| ID | Action | Effort | Owner | Rule(s) |
|---|---|---|---|---|
| P0-01 | **Remove `Loan_Account` (141 attrs) from CRR logical model** — file formal CR to reassign to `silver.lending` | S | Data Architect | SA Scoping |
| P0-02 | **Delete `reference_aggregated_pipelines` and `reference_aggregated_individualcontracts`** from Silver — pre-computed aggregations prohibited | S | Data Engineer | SLV-007 |
| P0-03 | **Populate `reference_bureau_source`** — currently empty; blocking referential integrity for all bureau score records | S | Data Engineer | Gate-3 RI |
| P0-04 | **Resolve duplicate entity**: deprecate `customer_riskrating_and_scoring` (5 KB) in favour of `customer_riskrating_scoring` (277 KB) or vice versa; document decision | S | Data Architect | NAM-003 |
| P0-05 | **Fix `Rating Valid Till` mapping** — must not map to `CustInfo.UpdatedOn`; source the actual model expiry date from risk model governance system | S | Data Steward + Data Engineer | Data Quality |
| P0-06 | **Fix `Loan Facility ID` mapping** in both `Customer_CollectionSummary` and `Customer_DeliquencyHistory` — `Product_Code` is a product type code, not a facility identifier | S | Data Steward + Data Engineer | SLV-004 |
| P0-07 | **Fix `Report Format` mapping** in `Reference_Bureau_Source` — `SCORE.Value` is a numeric score, not a report format | S | Data Steward | Data Quality |
| P0-08 | **Align schema to production catalog**: rename `silver_dev.customer_riskrating_scoring` → `silver.risk` (or register `customer_risk_rating` as a formal sub-schema) | M | Platform Engineering | NAM-002 |
| P0-09 | **Add `<entity>_sk` (MD5) + `<entity>_bk` to all 7 core entity tables** and update Silver ETL pipelines | M | Silver Pipeline Engineering | SLV-001, SLV-002 |

#### P1 — Next Sprint

| ID | Action | Effort | Owner | Rule(s) |
|---|---|---|---|---|
| P1-01 | **Implement SCD-2 columns** on all 7 core entities: add `effective_from`, `effective_to`, `is_current`, `run_id`, `source_ingestion_date`; remove `Business Start Date`/`Business End Date`/`Is Active Flag` | M | Silver Pipeline Engineering | SLV-003, §2.7 |
| P1-02 | **Fix column naming violations**: rename `IsLogicallyDeleted` → `is_deleted`, `Business Start Date` → `effective_from`, `Business End Date` → `effective_to`, `Is Active Flag` → `is_current` on all entities | S | Data Architect + Engineering | NAM-001 |
| P1-03 | **Correct entity name typo**: `Deliquency` → `Delinquency` in logical model, physical table (`customer_deliquency_history` → `customer_delinquency_history`), and all pipelines | S | Data Architect + Engineering | NAM-001 |
| P1-04 | **Define precise data types** for all 93 logical attributes — STRING is not acceptable for dates, scores, and flags. Minimum required: score values → DECIMAL(10,4); dates → DATE/TIMESTAMP; flags → BOOLEAN; codes → STRING | S | Data Architect | §2.7 |
| P1-05 | **Implement code canonicalization** via `silver.reference.code_mapping` for `Status`, `Account Type`, `Score Grade`, `Rating Score` | M | Data Engineer | SLV-004 |
| P1-06 | **Add DQ controls**: implement quarantine tables; add `data_quality_status`, `dq_issues` columns; enforce 5% error-rate threshold in pipelines | M | Platform Engineering | SLV-005, §3.2 |
| P1-07 | **Map `Delinquency Bucket` and `Resolved Flag`** in `Customer_DeliquencyHistory` — critical IFRS 9 inputs; both currently "Not Available" | M | Data Steward | IFRS 9 |
| P1-08 | **Map `Case Open Date`, `Current Bucket`, `Last Activity Date`** in `Customer_CollectionSummary` — all "Not Available"; entity will be non-functional without these | M | Data Steward | SLV-005 |
| P1-09 | **Register all 39 tables in data catalog** (Informatica GDGC) with business description, data steward, classification, and lineage | M | Data Steward + Platform | §2.12 |
| P1-10 | **Resolve governance question on `Customer_CollectionSummary` and `Customer_DeliquencyHistory` scope** — document whether these are canonical entities in CRR or read-only snapshots from `silver.lending.collections` | S | Data Architect + Lending SA Steward | SA Scoping |

#### P2 — Backlog

| ID | Action | Effort | Owner | Rule(s) |
|---|---|---|---|---|
| P2-01 | **Define `Customer_LoanLinkage` entity** — currently a shell with 0 attributes; determine if it is a bridge table or redundant | S | Data Architect | §2.11 |
| P2-02 | **Reclassify all 28 `reference_*` staging tables** — identify which belong in Bronze, which in `silver.reference`, which in `silver.party`/`silver.account` | L | Data Architect + Platform Engineering | SLV-006, §2.1 |
| P2-03 | **Add `AECB_Range` to logical model** in Erwin — currently physical-only; undocumented attribute | S | Data Architect | §2.11 |
| P2-04 | **Implement PII masking** for CIF/Customer Number fields via SHA-256 | M | Platform Engineering | §10 (PII) |
| P2-05 | **Document and justify** `Customer_CollectionSummary` and `Customer_DeliquencyHistory` scope in CRR if retained | S | Data Steward | SA Scoping |
| P2-06 | **Complete Field Classification** — all 210 attributes have null `Attribute_Desc`, `Source of change`, and `Source Systems` | L | Data Steward | §2.12 |
| P2-07 | **Investigate `reference_financialsummary` and `reference_bulkportfolio_characteristics`** for aggregation violations | S | Data Architect | SLV-007 |
| P2-08 | **Enforce UTC timestamp storage** in all Silver pipelines; add `CONVERT_TIMEZONE` calls | S | Data Engineer | SLV-009, §2.9 |

---

### 7.2 Suggested Remediation Schedule

| Sprint | P-Level | Key Deliverables |
|---|---|---|
| **Sprint 1** | P0 | Remove Loan_Account from logical model; delete aggregated tables; populate bureau_source; fix semantic mapping errors (Rating Valid Till, Report Format, Loan Facility ID); resolve duplicate entity |
| **Sprint 2** | P0 | Add SK/BK to all 7 core entities; align schema to `silver.risk`; fix entity name typo (Delinquency) |
| **Sprint 3** | P1 | Implement SCD-2 columns; fix naming violations; define precise data types |
| **Sprint 4** | P1 | Code canonicalization via `silver.reference.code_mapping`; DQ controls and quarantine tables |
| **Sprint 5** | P1 | Complete missing attribute mappings (DPD Bucket, Resolved Flag, Case Open Date, Current Bucket); data catalog registration |
| **Sprint 6+** | P2 | Reference table reclassification; PII masking; Field Classification completion; LoanLinkage entity definition |

---

## 8. Appendix

### A: Mapping Summary

| Entity | Total Attrs | Fully Mapped | Partially Mapped | Not Available | Mapping % | Data Type Specified |
|---|---|---|---|---|---|---|
| Reference_Risk_Model | 12 | 6 | 2 | 4 | 50% | 0% |
| Reference_Bureau_Source | 10 | 7 | 1 | 2 | 70% | 0% |
| Customer_RiskRating & Scoring | 14 | 9 | 1 | 4 | 64% | 0% |
| Customer_ScoreHistory | 12 | 8 | 1 | 3 | 67% | 0% |
| Customer_BureauScore | 13 | 10 | 1 | 2 | 77% | 0% |
| Customer_CollectionSummary | 16 | 7 | 2 | 7 | 44% | 0% |
| Customer_DeliquencyHistory | 16 | 7 | 2 | 7 | 44% | 0% |
| Customer_LoanLinkage | 0 | 0 | 0 | 0 | 0% | 0% |
| **TOTAL** | **93** | **54** | **10** | **29** | **58%** | **0%** |

**Note:** `Source System Code` and `Source System ID` are counted as "Mapped" in the original mapping document but have no source system, table, or column specified — these are effectively phantom mappings.

---

### B: Guideline Citations

| Guideline Reference | Rule Text | Relevance to This Assessment |
|---|---|---|
| `guidelines/06-naming-conventions.md` NAM-001 | "Use snake_case for all identifiers (catalogs, schemas, tables, columns, views). Use lowercase only." | Violated by `IsLogicallyDeleted`, `Business Start Date`, `Is Active Flag`, `CIF(Customer Number)` |
| `guidelines/06-naming-conventions.md` NAM-002 | "Silver: `silver.<subject_area>.<entity>`" | Physical schema `silver_dev.customer_riskrating_scoring` violates this pattern |
| `guidelines/06-naming-conventions.md` NAM-003 | "Silver entity: `<entity>`" | `reference_*` tables use non-standard naming for what should be Bronze or canonical reference entities |
| `guidelines/11-modeling-dos-and-donts.md` §2.1 | "Decompose entities into 3NF: every non-key attribute depends on the whole primary key and nothing but the primary key." | 28 Bronze-replica tables violate 3NF; account attributes transitive in collection/delinquency entities |
| `guidelines/11-modeling-dos-and-donts.md` §2.3 | "Every Silver entity table must have `effective_from`, `effective_to`, `is_current`." | All entities missing these columns |
| `guidelines/11-modeling-dos-and-donts.md` §2.4 | "Derive surrogate keys deterministically: `MD5(UPPER(TRIM(business_key)))`." | No surrogate keys on any entity |
| `guidelines/11-modeling-dos-and-donts.md` §2.5 | "Resolve all source-system codes to canonical platform values using `silver.reference.code_mapping`." | Raw codes stored in `Status`, `Account Type`, `Score Grade` |
| `guidelines/11-modeling-dos-and-donts.md` §2.6 | "Store only atomic, un-aggregated facts in Silver. Pre-compute totals, averages, counts, ratios, or any derived metrics are prohibited." | `reference_aggregated_pipelines` and `reference_aggregated_individualcontracts` violate this |
| `guidelines/11-modeling-dos-and-donts.md` §2.7 | "Every Silver entity table must include: `source_system_code`, `source_system_id`, `create_date`, `update_date`, `delete_date`, `is_active_flag`, `effective_from`, `effective_to`, `is_current`, `run_id`, `source_ingestion_date`." | 5 of 11 columns missing on all entities |
| `guidelines/01-data-quality-controls.md` §3 | "Failed records routed to quarantine / error tables rather than discarded. All DQ results must be logged and auditable." | No quarantine tables; no DQ status columns |
| `guidelines/01-data-quality-controls.md` §4 | "Time Zone Normalisation: All timestamps stored in UTC." | Cannot verify — all types are STRING in logical model |
| `guidelines/01-data-quality-controls.md` §5 | "Every Silver and Gold table must carry: `record_source`, `record_origin`, `record_hash`, `is_current`, `valid_from_at`, `valid_to_at`, `ingested_at`, `processed_at`, `batch_id`, `data_quality_status`, `dq_issues`." | None of these present under these canonical names |
| `sa/subject-areas.md` SA-17 | "Risk & Compliance: `silver.risk` — KYC Levels, AML Risk Scores, Watchlist Hits, Customer Risk Ratings, FATCA/CRS flags, and Basel RWA." | CRR is a sub-domain of `silver.risk`; standalone schema `customer_riskrating_scoring` is not aligned |

---

### C: Industry References

| Standard | Section | Relevance |
|---|---|---|
| **BIAN v11** | Service Domain SD-0131 "Customer Credit Rating" | Primary domain mapping for all CRR entities |
| **BIAN v11** | Service Domain SD-0120 "Customer Risk Profile" | Secondary domain — AML risk score, KYC risk band |
| **BCBS 239** | Principles 2, 3 (Data Architecture & IT Infrastructure), Principle 6 (Completeness) | DPD bucket and Delinquency History are regulatory CDEs requiring complete lineage |
| **IFRS 9** | ECL Staging Framework — Paragraph 5.5 | `Customer_DeliquencyHistory.Delinquency_Bucket` is a direct IFRS 9 Stage 2/3 trigger attribute; must be sourced and complete |
| **IFRS 9** | Paragraph B5.5.17 (30 DPD backstop) | `Days_Past_Due` must be numerically typed (INTEGER) and UTC-timestamped for the 30-DPD backstop rule |
| **CBUAE** | Credit Risk Guidelines (Circular 28/2010) | Bureau scores from AECB are mandatory inputs to UAE Central Bank credit risk assessments; `customer_bureauscore` is critical |
| **FATF / UAE AML** | FATF Rec. 10–12 | Customer risk rating is a key AML control output; incomplete ratings violate CDD requirements |
| **IFW (IBM/Teradata)** | Risk Subject Area — Customer Credit Risk pattern | Confirms appropriate entities: Risk Rating, Bureau Score, Delinquency History; no aggregation at Silver layer |

---

*End of Report — Customer Risk Rating Gap Assessment — 2026-04-27*
