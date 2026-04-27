# Finance GL Subject Area — Silver Layer Gap Assessment Report

**Subject Area:** Finance GL (`silver.gl`)
**Assessment Date:** 2026-04-27
**Assessor:** Senior Data Platform Architect (Automated via Logical Model Assessor)
**Model Version:** Bank Logical Model v1.0 (December 2025)
**Priority (from subject-areas.md):** 1 — Critical
**Schema:** `silver.gl`

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

| Dimension | Status | Detail |
|---|---|---|
| **Overall Readiness** | 🔴 Not Production-Ready | Critical structural, mapping, and governance gaps throughout |
| **Entities Assessed** | 7 | All logical entities assessed |
| **Physical Tables** | 7 | Present in `silver_dev.gl` |
| **P0 Findings** | 14 | Require immediate resolution before any production promotion |
| **P1 Findings** | 18 | Next sprint |
| **P2 Findings** | 9 | Backlog |
| **Total Findings** | 41 | |
| **Missing Critical Entities** | 3 | GL_PERIOD, GL_BALANCE (Trial Balance), GL_ACCOUNT_HIERARCHY |
| **Surrogate Key Compliance** | 🔴 0 / 7 entities | No `<entity>_sk` or `<entity>_bk` on any entity |
| **SCD-2 Compliance** | 🔴 0 / 7 entities | No SCD-2 columns present in any mapping |
| **Metadata Column Compliance** | 🔴 0 / 7 entities | Mandatory audit columns absent from all entities |
| **Monetary Type Compliance** | 🔴 0 / 2 monetary entities | All amounts typed as STRING instead of DECIMAL(18,4) |
| **Critical Mapping Errors** | 🔴 3 entities | Confirmed wrong-source-column mappings in GL_ADJUSTMENTS, PRODUCT_GL_MAPPING, CENTRAL_BANK_CODES |
| **Subject Area Scoping** | 🟡 Partial | 2 entities (VAT_RULES, CENTRAL_BANK_CODES) belong in `silver.reference` |
| **Catalog Descriptions** | 🔴 0 / 7 tables | All physical tables have `null` description in Unity Catalog |

### 1.2 Top 5 Immediate Actions

| # | Action | Priority | Effort | Owner |
|---|---|---|---|---|
| 1 | Add `<entity>_sk` (MD5 surrogate) and `<entity>_bk` to every entity | P0 | Large | Data Modeling |
| 2 | Fix three critical wrong-source-column mappings: `adjustment_id`, `gl_posting_rule`, `central_bank_code` | P0 | Medium | ETL / Data Modeling |
| 3 | Add all mandatory SCD-2 and audit columns to every entity (`effective_from`, `effective_to`, `is_current`, `run_id`, `source_ingestion_date`, etc.) | P0 | Large | ETL / DBA |
| 4 | Define and implement the three missing critical entities: GL_PERIOD, GL_BALANCE, GL_ACCOUNT_HIERARCHY | P0 | Large | Data Modeling |
| 5 | Correct all monetary amount data types from STRING to DECIMAL(18,4); align with fils convention for AED | P0 | Medium | Data Modeling / ETL |

---

## 2. Subject Area Architecture Assessment

### 2.1 Scoping vs. `sa/subject-areas.md`

**Description per subject-areas.md:** "The Books: Chart of Accounts, General Ledger, Trial Balances, and Financial Period definitions. Full Balance Sheet and P&L coverage."

**Verdict:** 🟡 Partially Correct with two scoping violations.

| Scoping Issue | Entity | Finding |
|---|---|---|
| ✅ In-scope | `GL_MASTER`, `GL_TRANSACTIONS`, `GL_ADJUSTMENTS`, `GL_RECONCILIATION` | Correctly scoped to Finance GL |
| 🟡 Borderline | `PRODUCT_GL_MAPPING` | Cross-SA mapping table (links Product SA to GL SA); belongs in `silver.gl` only if GL is the authoritative publisher of posting rules |
| 🔴 Misplaced | `VAT_RULES` | UAE VAT rate reference data — belongs in `silver.reference`, not `silver.gl` |
| 🔴 Misplaced | `CENTRAL_BANK_CODES` | CBUAE reporting category master — belongs in `silver.reference`, not `silver.gl` |
| 🔴 Missing | `GL_PERIOD` / Fiscal Calendar | Described in subject-areas.md but absent from model and physical tables |
| 🔴 Missing | `GL_BALANCE` / Trial Balance | Described in subject-areas.md ("Trial Balances") but absent from model and physical tables |
| 🔴 Missing | `GL_ACCOUNT_HIERARCHY` | `gl_parent_account_id` exists in logical model for GL_MASTER but no hierarchy/CoA tree entity is defined |

### 2.2 Domain Decomposition vs. BIAN / IFW

**BIAN Service Domain:** Financial Accounting → General Ledger Management (SD-186)

Under BIAN and IFW, the Finance GL domain requires:
- **Chart of Accounts** — account master with hierarchy (present as GL_MASTER, but hierarchy is incomplete)
- **GL Journal Entry / Posting** — transactional double-entry ledger (present as GL_TRANSACTIONS, but lacks debit/credit split)
- **GL Balance / Period Balance** — period-end account balances (⚠️ absent)
- **Fiscal Period / Accounting Calendar** — period definitions (⚠️ absent)
- **GL Reconciliation** — sub-ledger to GL tie-out (present as GL_RECONCILIATION)
- **GL Adjustment / Manual Entry** — manual corrections (present as GL_ADJUSTMENTS)

The current model covers 4 of 6 BIAN-required components. Critically, **GL_BALANCE** and **GL_PERIOD** are missing — without these, it is impossible to produce a Trial Balance or any period-end financial statement, which directly contradicts the stated goal of "Full Balance Sheet and P&L coverage."

Additionally, GL_TRANSACTIONS stores a "signed amount" approach rather than explicit debit/credit columns. While acceptable if combined with a strict DR=+ / CR=- convention, this convention is not formally documented and creates ambiguity when integrating multiple source systems (FINACLE, PRIME, RLS) that may use different sign conventions.

### 2.3 Identity Strategy (SLV-001/002)

**Verdict:** 🔴 Non-Compliant — No surrogate keys or business keys on any entity.

Rule SLV-001 requires `<entity>_sk` (MD5 deterministic surrogate) on every entity. Rule SLV-002 requires `<entity>_bk` (natural/business key) on every entity. Neither column pattern appears in the logical model attributes, the data mapping, or can be inferred from the physical structures CSV (which contains no column-level DDL).

| Entity | Expected SK | Expected BK | Status |
|---|---|---|---|
| GL_MASTER | `gl_master_sk` | `gl_account_id` (candidate) | 🔴 Missing both |
| GL_TRANSACTIONS | `gl_transaction_sk` | `transaction_id` (candidate) | 🔴 Missing both |
| GL_RECONCILIATION | `gl_reconciliation_sk` | `reconciliation_id` (candidate) | 🔴 Missing both |
| GL_ADJUSTMENTS | `gl_adjustment_sk` | `adjustment_id` (candidate) | 🔴 Missing both |
| PRODUCT_GL_MAPPING | `product_gl_mapping_sk` | composite BK (candidate) | 🔴 Missing both |
| VAT_RULES | `vat_rule_sk` | `gl_account_id` (candidate) | 🔴 Missing both |
| CENTRAL_BANK_CODES | `central_bank_code_sk` | `central_bank_code` (candidate) | 🔴 Missing both |

### 2.4 SCD Strategy (SLV-003)

**Verdict:** 🔴 Non-Compliant — No SCD-2 columns on any entity in the mapping or logical model.

The logical model lists `Business Start Date`, `Business End Date`, and `Is Active Flag` as attributes in all entities. However:
- These use space-separated names (not snake_case — NAM-001 violation)
- They are typed as `CHAR(18)` (incorrect; dates should be `DATE` or `TIMESTAMP`)
- They are NOT mapped to any source system columns
- The proper canonical names per guideline 11 are `effective_from` (TIMESTAMP), `effective_to` (TIMESTAMP), and `is_current` (BOOLEAN)

Without SCD-2 columns, there is no change history, no point-in-time query capability, and no audit trail for regulatory compliance.

### 2.5 Cross-Entity Referential Integrity

**Verdict:** 🔴 Critical Gap — FK integrity between GL_MASTER and GL_TRANSACTIONS is broken at the source level.

| FK Relationship | Mapping Evidence | Issue |
|---|---|---|
| `GL_TRANSACTIONS.gl_account_id` → `GL_MASTER.gl_account_id` | FINACLE: ACID (VARCHAR2, 11-char account ID) vs GL_SUB_HEAD_CODE (VARCHAR2(5), GL code) | Different source columns with different semantics and lengths — FK will not join |
| `GL_RECONCILIATION.transaction_id` → `GL_TRANSACTIONS.transaction_id` | Both from HIST_TRAN_DTL_TABLE.TRAN_ID | Plausible, but reconciliation_id itself is unmapped |
| `GL_ADJUSTMENTS.gl_account_id` → `GL_MASTER.gl_account_id` | Same ACID vs GL_SUB_HEAD_CODE mismatch as GL_TRANSACTIONS | FK will not join |
| `PRODUCT_GL_MAPPING.gl_account_id` → `GL_MASTER.gl_account_id` | ACID / EQUATION_ACCT_CODE (13 digits) vs GL_SUB_HEAD_CODE (5 digits) | FK mismatch |
| `VAT_RULES.gl_account_id` → `GL_MASTER.gl_account_id` | GL_CODE from same source GL_SUB_HEAD_TABLE | Plausible join on GL_SUB_HEAD_TABLE.GL_CODE, but mapping uses GL_CODE not GL_SUB_HEAD_CODE for GL_MASTER — inconsistency |

### 2.6 ETL Mapping Completeness and Lineage

**Verdict:** 🔴 Incomplete — Multiple attributes unmapped, several critically wrong.

| Entity | Total Logical Attributes | Mapped | Unmapped | Wrong Mapping |
|---|---|---|---|---|
| GL_MASTER | 15 | 3 (20%) | 12 (80%) | 0 confirmed |
| GL_TRANSACTIONS | 18 | 9 (50%) | 9 (50%) | 2 confirmed (date inversion) |
| GL_RECONCILIATION | 16 | 4 (25%) | 12 (75%) | 0 confirmed |
| GL_ADJUSTMENTS | 15 | 4 (27%) | 11 (73%) | 1 confirmed (adjustment_id) |
| PRODUCT_GL_MAPPING | 17 | 6 (35%) | 11 (65%) | 2 confirmed (gl_posting_rule, central_bank_code) |
| VAT_RULES | 13 | 3 (23%) | 10 (77%) | 1 suspected (vat_rate source) |
| CENTRAL_BANK_CODES | 9 | 2 (22%) | 7 (78%) | 1 confirmed (central_bank_code) |

**Overall mapping completeness: ~29% across the subject area.** The remaining 71% of logical attributes have no confirmed source mapping.

---

## 3. Entity Inventory

| # | Logical Entity | Physical Table | Schema | Grain | Mapping % | SK Present | SCD-2 | Metadata | Overall |
|---|---|---|---|---|---|---|---|---|---|
| 1 | GL_MASTER | `gl_master` | `silver_dev.gl` | One row per GL account code | ~20% | 🔴 No | 🔴 No | 🔴 No | 🔴 Critical |
| 2 | GL_TRANSACTIONS | `gl_transaction` | `silver_dev.gl` | One row per GL posting line | ~50% | 🔴 No | 🔴 No | 🔴 No | 🔴 Critical |
| 3 | GL_RECONCILIATION | `gl_reconciliation` | `silver_dev.gl` | One row per recon event | ~25% | 🔴 No | 🔴 No | 🔴 No | 🔴 Critical |
| 4 | GL_ADJUSTMENTS | `gl_adjustment` | `silver_dev.gl` | One row per adjustment | ~27% | 🔴 No | 🔴 No | 🔴 No | 🔴 Critical |
| 5 | PRODUCT_GL_MAPPING | `product_gl_mapping` | `silver_dev.gl` | One row per product/txn-type/GL account combination | ~35% | 🔴 No | 🔴 No | 🔴 No | 🔴 Critical |
| 6 | VAT_RULES | `vat_rules` | `silver_dev.gl` | One row per GL account VAT rule | ~23% | 🔴 No | 🔴 No | 🔴 No | 🔴 Critical |
| 7 | CENTRAL_BANK_CODES | `central_bank_code` | `silver_dev.gl` | One row per CBUAE reporting code | ~22% | 🔴 No | 🔴 No | 🔴 No | 🔴 Critical |
| — | GL_PERIOD | *(absent)* | — | One row per fiscal period | 0% | — | — | — | 🔴 Missing |
| — | GL_BALANCE | *(absent)* | — | One row per account per period | 0% | — | — | — | 🔴 Missing |
| — | GL_ACCOUNT_HIERARCHY | *(absent)* | — | One row per hierarchy node | 0% | — | — | — | 🔴 Missing |

---

## 4. Per-Entity Assessments

### 4.1 GL_MASTER

**Physical Table:** `silver_dev.gl.gl_master`
**Logical Definition:** General Ledger Chart of Accounts
**Grain:** One row per unique GL account code

#### 4.1.1 Industry Fitness

GL_MASTER serves as the Chart of Accounts (CoA). In BIAN SD-186 and IFW, the CoA entity is responsible for the classification hierarchy of all bookable accounts. The current entity conflates:
- Leaf-level GL accounts (atomic bookable nodes)
- Parent/hierarchy relationships (via `gl_parent_account_id`)

This mix is acceptable as a single table with a self-referencing FK, but the hierarchy is incomplete (only one level of `gl_parent_account_id` with no `gl_level`, `gl_path`, or full hierarchy table). For balance sheet and P&L roll-ups, a recursive hierarchy or bridge table is required.

The entity is correctly scoped to the GL SA. The `product_type` attribute (PERSONAL, BUSINESS, TRADE) introduces a product-dimension concern into the CoA master — this should be a FK to `silver.product` rather than a denormalized classification code.

#### 4.1.2 Attribute-Level Review

| Attribute (Logical) | Logical Type | Mapping Status | Physical Source | Issues |
|---|---|---|---|---|
| `gl_account_id` | STRING | Mapped | FINACLE: GL_SUB_HEAD_CODE (VARCHAR2(5)) / PRIME: CODE (CHAR, 13-digit) / RLS: EQUATION_ACCT_CODE (VARCHAR2, 13-digit) | 🔴 Source length mismatch (5 vs 13 chars) — FK integrity broken; no reconciliation logic |
| `gl_account_name` | STRING | Mapped (FINACLE only) | GL_SUB_HEAD_DESC (VARCHAR2(30)) | 🟡 Max 30 chars from source may truncate long names; PRIME/RLS not mapped |
| `gl_category` | STRING | Mapped (FINACLE only) | GL_CLASSIFICATION_FLG (CHAR) | 🟡 Raw source flag stored; must use `silver.reference.code_mapping` per SLV-004 |
| `product_type` | STRING | Unmapped | No source | 🔴 Unmapped; crosses SA boundary (should be FK to Product SA) |
| `is_reconcilable` | STRING | Unmapped | No source | 🔴 Unmapped; should be BOOLEAN not STRING |
| `Source System Code` | STRING | Unmapped | No source | 🔴 NAM-001 violation (spaces); should be `source_system_code` |
| `Source System Id` | STRING | Unmapped | No source | 🔴 NAM-001 violation (spaces); should be `source_system_id` |
| `IsLogicallyDeleted` | STRING | Unmapped | No source | 🔴 NAM-001 violation (PascalCase); should be `is_logically_deleted` BOOLEAN |
| `Create Date` | STRING | Unmapped | No source | 🔴 NAM-001 (spaces); type STRING should be TIMESTAMP; should be `create_date` |
| `Update Date` | STRING | Unmapped | No source | 🔴 NAM-001 (spaces); type STRING should be TIMESTAMP; should be `update_date` |
| `Deleted Date` | STRING | Unmapped | No source | 🔴 NAM-001 (spaces); type STRING should be TIMESTAMP; should be `delete_date` |
| `gl_parent_Account_id` | CHAR(18) | Unmapped | No source | 🔴 NAM-001 (mixed case); CHAR(18) incorrect; should be STRING matching gl_account_id type |
| `Business Start Date` | CHAR(18) | Unmapped | No source | 🔴 NAM-001 (spaces); CHAR(18) incorrect; should be `effective_from` TIMESTAMP |
| `Business End Date` | CHAR(18) | Unmapped | No source | 🔴 NAM-001 (spaces); CHAR(18) incorrect; should be `effective_to` TIMESTAMP |
| `Is Active Flag` | CHAR(18) | Unmapped | No source | 🔴 NAM-001 (spaces); CHAR(18) incorrect; should be `is_current` BOOLEAN |
| `gl_master_sk` | — | Missing | — | 🔴 SLV-001: surrogate key absent |
| `gl_master_bk` | — | Missing | — | 🔴 SLV-002: business key absent |

#### 4.1.3 Metadata Completeness (SLV-008)

| Required Column | Present in Mapping | Notes |
|---|---|---|
| `record_source` / `source_system_code` | 🔴 No | Logical model has `Source System Code` (wrong name, unmapped) |
| `record_origin` / `source_system_id` | 🔴 No | Logical model has `Source System Id` (wrong name, unmapped) |
| `record_hash` | 🔴 No | Absent |
| `is_current` | 🔴 No | Logical model has `Is Active Flag` (wrong name, wrong type, unmapped) |
| `valid_from_at` / `effective_from` | 🔴 No | Logical model has `Business Start Date` (wrong name, wrong type, unmapped) |
| `valid_to_at` / `effective_to` | 🔴 No | Logical model has `Business End Date` (wrong name, wrong type, unmapped) |
| `ingested_at` / `source_ingestion_date` | 🔴 No | Absent |
| `processed_at` / `create_date` | 🔴 No | Logical model has `Create Date` (wrong name, unmapped) |
| `batch_id` / `run_id` | 🔴 No | Absent |
| `data_quality_status` | 🔴 No | Absent |
| `dq_issues` / `dq_flags` | 🔴 No | Absent |

**All 11 mandatory metadata columns are absent or incorrectly named.**

---

### Finding GL-MASTER-001 — Missing Surrogate and Business Key

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | `SLV-001` — "Every Silver entity must have `<entity>_sk` (MD5 surrogate key)" and `SLV-002` — "Every Silver entity must have `<entity>_bk` (business/natural key)" |
| Evidence | Logical model: GL_MASTER has no `gl_master_sk` or `gl_master_bk` attribute · Physical: CSV has no column DDL to confirm |
| Affected Table | `silver.gl.gl_master` |
| Affected Column(s) | `gl_master_sk` (missing), `gl_master_bk` (missing) |
| Confidence | 0.98 |

**Description:** No surrogate key (`gl_master_sk` = MD5 of business key) and no explicit business key column (`gl_master_bk`) are defined. The `gl_account_id` attribute is a candidate BK but is not formally designated. Without a stable SK, downstream Gold joins will be non-deterministic and cross-source reconciliation is impossible.

**Remediation:** Add `gl_master_sk STRING NOT NULL` (computed as `MD5(UPPER(TRIM(gl_account_id)))`) and `gl_master_bk STRING NOT NULL` (assigned from `gl_account_id` after source code canonicalization). Both must be NOT NULL and `gl_master_sk` must be the PK.

**Estimated Effort:** Small
**Owner:** Data Modeling

---

### Finding GL-MASTER-002 — GL Account ID Source Length Mismatch (FK Integrity Broken)

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | `SLV-005` — "DQ-gated writes; quarantine table" (referential integrity gate); guideline 11 §2.5 "Canonicalization" |
| Evidence | Mapping: FINACLE GL_SUB_HEAD_CODE VARCHAR2(5) vs PRIME CODE CHAR(13-digit) vs RLS EQUATION_ACCT_CODE VARCHAR2(13-digit). GL_TRANSACTIONS and GL_ADJUSTMENTS map gl_account_id from ACID (VARCHAR2(11), account number) — entirely different semantic object |
| Affected Table | `silver.gl.gl_master`, `silver.gl.gl_transaction`, `silver.gl.gl_adjustment` |
| Affected Column(s) | `gl_account_id` |
| Confidence | 0.95 |

**Description:** `GL_MASTER.gl_account_id` is sourced from `GL_SUB_HEAD_CODE` (5-character GL sub-head code) in FINACLE, but `GL_TRANSACTIONS.gl_account_id` and `GL_ADJUSTMENTS.gl_account_id` are sourced from `ACID` (11-character account number) in FINACLE — which is an account identifier, NOT a GL account code. The FK relationship `GL_TRANSACTIONS.gl_account_id → GL_MASTER.gl_account_id` will produce zero matches on Finacle data. PRIME and RLS use 13-digit codes, adding a third incompatible format.

**Remediation:** Establish a cross-source GL account code mapping. GL_MASTER's PK should be the canonical platform GL account code. A separate mapping table should resolve source-system GL codes to canonical codes, with transformation logic documented per source system. GL_TRANSACTIONS should store the canonical code, not the source ACID.

**Estimated Effort:** Large
**Owner:** Data Modeling / ETL

---

### Finding GL-MASTER-003 — All Metadata and SCD Columns Missing

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | `SLV-008` — "All metadata columns present"; `SLV-003` — "SCD-2 default" |
| Evidence | Logical model: Business Start/End Date as CHAR(18) unmapped; no `effective_from`, `effective_to`, `is_current`, `run_id`, `source_ingestion_date`, `record_hash`, `data_quality_status`, `dq_issues` |
| Affected Table | `silver.gl.gl_master` |
| Affected Column(s) | All 11 mandatory metadata columns |
| Confidence | 0.99 |

**Description:** All 11 mandatory Silver metadata columns are absent from GL_MASTER's mapping. The logical model contains placeholder attributes (`Business Start Date`, `Business End Date`, `Is Active Flag`) with wrong types (CHAR(18)) and wrong names (spaces instead of snake_case) that have no source mapping. The CoA is a slowly-changing dimension — account hierarchies, names, and categories change over time and must be tracked via SCD-2.

**Remediation:** Add all required audit/SCD columns with correct types and names as per guideline 11 §2.7: `source_system_code STRING NOT NULL`, `source_system_id STRING`, `create_date TIMESTAMP NOT NULL`, `update_date TIMESTAMP`, `delete_date TIMESTAMP`, `is_active_flag STRING`, `effective_from TIMESTAMP NOT NULL`, `effective_to TIMESTAMP`, `is_current BOOLEAN NOT NULL`, `run_id STRING`, `source_ingestion_date TIMESTAMP`. Remove the incorrectly-named logical model placeholders.

**Estimated Effort:** Medium
**Owner:** Data Modeling / ETL

---

### Finding GL-MASTER-004 — Raw Source Category Code Stored Without Mapping (SLV-004)

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | Medium |
| Guideline Rule | `SLV-004` — "No source-system codes; use `silver.reference.code_mapping`" |
| Evidence | Mapping: gl_category ← FINACLE GL_CLASSIFICATION_FLG (CHAR); definition says "ASSET, LIABILITY, INCOME, EXPENSE from ERP" — these are source values, not canonical codes |
| Affected Table | `silver.gl.gl_master` |
| Affected Column(s) | `gl_category` |
| Confidence | 0.90 |

**Description:** `gl_category` stores the raw source-system classification flag from Finacle GL_CLASSIFICATION_FLG. While the values (ASSET/LIABILITY/INCOME/EXPENSE) appear to be standard accounting categories, they are source-native representations that may differ across PRIME and RLS (neither is mapped for this field). The canonical mapping must be registered in `silver.reference.code_mapping`.

**Remediation:** Join against `silver.reference.code_mapping` to resolve source classification flags to canonical `gl_category_code` values. Register mappings for all three source systems. Rename column to `gl_category_code` per naming conventions.

**Estimated Effort:** Small
**Owner:** ETL / Governance

---

### Finding GL-MASTER-005 — Naming Convention Violations Throughout Logical Model

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | Medium |
| Guideline Rule | `NAM-001` — "Use snake_case for all identifiers; lowercase only; no spaces" |
| Evidence | Logical model attributes: `Source System Code`, `Source System Id`, `IsLogicallyDeleted`, `Create Date`, `Update Date`, `Deleted Date`, `gl_parent_Account_id`, `Business Start Date`, `Business End Date`, `Is Active Flag` |
| Affected Table | `silver.gl.gl_master` |
| Affected Column(s) | 10 attributes with naming violations |
| Confidence | 0.99 |

**Description:** 10 of 15 logical attributes violate NAM-001 through use of spaces, PascalCase, or mixed case. These will propagate to DDL if not corrected during physical model generation.

**Remediation:** Rename all affected attributes: `Source System Code` → `source_system_code`, `Source System Id` → `source_system_id`, `IsLogicallyDeleted` → `is_logically_deleted`, `Create Date` → `create_date`, `Update Date` → `update_date`, `Deleted Date` → `delete_date`, `gl_parent_Account_id` → `gl_parent_account_id`, `Business Start Date` → `effective_from`, `Business End Date` → `effective_to`, `Is Active Flag` → `is_current`.

**Estimated Effort:** Small
**Owner:** Data Modeling

---

### Finding GL-MASTER-006 — `product_type` Crosses Subject-Area Boundary

| Field | Value |
|---|---|
| Priority | P2 |
| Criticality | Low |
| Guideline Rule | `SLV-006` — "3NF; no transitive dependencies"; guideline 11 §2.2 "Subject-Area Isolation" |
| Evidence | Logical model: `product_type` definition "PERSONAL, BUSINESS, TRADE from Mapping Tables" — product classification does not belong in GL |
| Affected Table | `silver.gl.gl_master` |
| Affected Column(s) | `product_type` |
| Confidence | 0.85 |

**Description:** `product_type` introduces product dimension attributes into the Chart of Accounts master, creating a transitive dependency (GL account → product type instead of GL account → GL account attributes). This should be a foreign key to `silver.product` or resolved via `PRODUCT_GL_MAPPING`.

**Remediation:** Remove `product_type` from GL_MASTER. Retrieve product classification from `PRODUCT_GL_MAPPING.product_code` → `silver.product`. If a denormalized product type is operationally required, document it in the Denormalization Register with justification.

**Estimated Effort:** Small
**Owner:** Data Modeling

---

### Finding GL-MASTER-007 — `is_reconcilable` Incorrectly Typed as STRING

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | Medium |
| Guideline Rule | `NAM-001` — "is_ prefix → BOOLEAN type"; guideline 11 §2.7 |
| Evidence | Logical model: `is_reconcilable STRING`; definition "Whether account requires reconciliation" is clearly a Boolean |
| Affected Table | `silver.gl.gl_master` |
| Affected Column(s) | `is_reconcilable` |
| Confidence | 0.97 |

**Description:** Boolean indicator `is_reconcilable` is declared as STRING in the logical model. Attributes with the `is_` prefix must be BOOLEAN per naming conventions. Storing Boolean flags as STRING creates ambiguity ('Y'/'N', 'TRUE'/'FALSE', '1'/'0') and prevents efficient predicate pushdown.

**Remediation:** Change logical and physical type to BOOLEAN. Add source mapping from the appropriate source column.

**Estimated Effort:** Small
**Owner:** Data Modeling

---

### 4.2 GL_TRANSACTIONS

**Physical Table:** `silver_dev.gl.gl_transaction`
**Logical Definition:** Atomic GL postings/journal entries
**Grain:** One row per GL posting line (single leg of a journal entry)

#### 4.2.1 Industry Fitness

GL journal entries are the atomic unit of double-entry bookkeeping. Each posting should represent a single debit or credit leg tied to a specific GL account. The current entity uses a signed-amount approach (`amount`, where DR=+ and CR=−), which is a valid convention but creates ambiguity when integrating multi-source systems that may use different sign conventions (Finacle, PRIME, RLS). Additionally:

- The entity contains `Account_number` and `Agreement_Id` — account and contract identifiers from operational systems. These are cross-SA references to `silver.account` and `silver.contract`. While keeping them as business keys for lineage purposes is acceptable, they should not drive GL posting logic in the Silver layer.
- Mixing `transaction_date` and `value_date` semantics: The mapping inverts these (see Finding GL-TRANS-002).
- There is no `journal_entry_id` or `batch_id` to group related debit/credit legs of the same journal entry, making balanced entry verification impossible.

#### 4.2.2 Attribute-Level Review

| Attribute (Logical) | Logical Type | Mapping Status | Physical Source | Issues |
|---|---|---|---|---|
| `transaction_id` | STRING | Mapped (FINACLE) | HIST_TRAN_DTL_TABLE.TRAN_ID (NUMBER(22)) | 🟡 Type mismatch: source is NUMBER(22), target is STRING — implicit cast, potential leading-zero loss |
| `source_transaction_id` | STRING | Unmapped | No source in mapping | 🟡 Mapping sheet shows this in logical model but not in Silver mapping; `source_reference` in mapping appears to be its replacement |
| `gl_account_id` | STRING | Mapped (FINACLE: ACID) | HIST_TRAN_DTL_TABLE.ACID (VARCHAR2, 11-char) | 🔴 ACID is an account ID, not a GL account code — FK to GL_MASTER broken |
| `amount` | STRING | Mapped (FINACLE: SUM_BAL_AMT) | CUSTOM.GL_AVG_BAL_TBL.SUM_BAL_AMT (NUMBER(30,4)) | 🔴 Type is STRING — must be DECIMAL(18,4) per SLV-010; source is average balance not a transaction amount |
| `currency code` | STRING | Mapped (FINACLE) | HIST_TRAN_DTL_TABLE.CRNCY_CODE (VARCHAR2) | 🔴 NAM-001: space in column name; should be `currency_code` |
| `transaction_date` | STRING | Mapped (FINACLE: VALUE_DATE) | HIST_TRAN_DTL_TABLE.VALUE_DATE (DATE) | 🔴 INVERTED: `transaction_date` should be accounting/posting date (PSTD_DATE), not value date; type STRING must be DATE |
| `value_date` | STRING | Mapped (FINACLE: PSTD_DATE) | HIST_TRAN_DTL_TABLE.PSTD_DATE (DATE) | 🔴 INVERTED: `value_date` (settlement) should be VALUE_DATE, not posting date; type STRING must be DATE |
| `source_system` | STRING | Unmapped | No source | 🔴 Unmapped; critical for multi-source GL |
| `source_reference` | STRING | Mapped (FINACLE: TRAN_ID) | HIST_TRAN_DTL_TABLE.TRAN_ID (NUMBER(22)) | 🟡 Same source as `transaction_id` — duplicate mapping |
| `Source_transaction_host_reference` | STRING | Unmapped | No source | 🔴 NAM-001 (PascalCase prefix) |
| `Account_number` | STRING | Mapped (multi-source) | GENERAL_ACCT_MAST_TABLE.ACCT_NUM | 🟡 NAM-001 (mixed case); cross-SA reference; should be `account_bk` |
| `Agreement_Id` | STRING | Mapped (multi-source) | GENERAL_ACCT_MAST_TABLE.FORACID | 🟡 NAM-001 (mixed case); cross-SA reference; should be `contract_bk` |
| `gl_transaction_sk` | — | Missing | — | 🔴 SLV-001 |
| `gl_transaction_bk` | — | Missing | — | 🔴 SLV-002 |
| Metadata columns (11) | — | Missing | — | 🔴 SLV-008 |

#### 4.2.3 Metadata Completeness

All 11 mandatory metadata columns absent (same pattern as GL_MASTER — see Finding GL-MASTER-003).

---

### Finding GL-TRANS-001 — Missing Surrogate and Business Key

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | `SLV-001`, `SLV-002` |
| Evidence | Logical model: no `gl_transaction_sk` or `gl_transaction_bk` attribute |
| Affected Table | `silver.gl.gl_transaction` |
| Affected Column(s) | `gl_transaction_sk` (missing), `gl_transaction_bk` (missing) |
| Confidence | 0.98 |

**Description:** No SK or BK present. `transaction_id` is the candidate BK but must be made explicit. For a multi-source GL (FINACLE, PRIME, RLS), the BK must include a source-system prefix to guarantee uniqueness: `MD5(CONCAT_WS('|', source_system, transaction_id))`.

**Remediation:** Add `gl_transaction_sk STRING NOT NULL` (MD5 of `source_system || '|' || transaction_id`) and `gl_transaction_bk STRING NOT NULL`. Ensure BK is unique per posting line.

**Estimated Effort:** Small
**Owner:** Data Modeling

---

### Finding GL-TRANS-002 — `transaction_date` and `value_date` Are Inverted

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | Guideline 11 §2.1 "correctness"; SLV-005 "DQ-gated writes" |
| Evidence | Mapping row 1: `transaction_date` ← HIST_TRAN_DTL_TABLE.VALUE_DATE (DATE); Mapping row 2: `value_date` ← HIST_TRAN_DTL_TABLE.PSTD_DATE (DATE). By banking definition, `transaction_date`/`accounting_date` = posting date (PSTD_DATE); `value_date` = settlement/value date (VALUE_DATE). |
| Affected Table | `silver.gl.gl_transaction` |
| Affected Column(s) | `transaction_date`, `value_date` |
| Confidence | 0.93 |

**Description:** The mapping inverts the two date columns. `transaction_date` (accounting/posting date) is mapped from `VALUE_DATE` (settlement date) and `value_date` is mapped from `PSTD_DATE` (posting date). This is a semantic inversion that will corrupt any financial period-end analysis: period cut-offs use accounting date, not value date, and the wrong cut-off will misstate P&L and Balance Sheet for cross-period transactions. Additionally, both columns are typed as STRING in the logical model instead of DATE.

**Remediation:** Correct mappings: `transaction_date` ← `PSTD_DATE` (posting/accounting date); `value_date` ← `VALUE_DATE` (settlement date). Change both column types to DATE (not TIMESTAMP, as these are date-precision fields).

**Estimated Effort:** Small
**Owner:** ETL

---

### Finding GL-TRANS-003 — `amount` Typed as STRING and Sourced from Average Balance Table

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | `SLV-010` — "Monetary amounts in smallest unit (fils for AED)"; guideline §2 Gate 2 "Monetary columns must use DECIMAL(18,4)" |
| Evidence | Logical model: `amount STRING`; Mapping: amount ← CUSTOM.GL_AVG_BAL_TBL.SUM_BAL_AMT (NUMBER(30,4)). SUM_BAL_AMT is an average balance total, not a transaction posting amount |
| Affected Table | `silver.gl.gl_transaction` |
| Affected Column(s) | `amount` |
| Confidence | 0.92 |

**Description:** Two compounded problems: (1) The amount column is typed as STRING in the logical model, violating the DECIMAL(18,4) mandate for all monetary columns. (2) The source column is `CUSTOM.GL_AVG_BAL_TBL.SUM_BAL_AMT` — an average balance summary field — rather than a transaction posting amount from a journal entry table. Average balances must never be stored as individual transaction amounts in Silver. The correct source for transaction amounts is `HIST_TRAN_DTL_TABLE.TRAN_AMT` (as correctly used in GL_ADJUSTMENTS).

**Remediation:** (1) Change logical and physical type to `DECIMAL(18,4)`. (2) Correct source mapping to `TBAADM.HIST_TRAN_DTL_TABLE.TRAN_AMT`. (3) For AED amounts, consider storing in fils (smallest unit) as BIGINT per SLV-010, with a companion `amount_currency_code` column.

**Estimated Effort:** Medium
**Owner:** ETL / Data Modeling

---

### Finding GL-TRANS-004 — `gl_account_id` Maps to Account ID (ACID), Not GL Account Code

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | Guideline 11 §2.5 "Canonicalization"; `SLV-005` referential integrity |
| Evidence | Mapping: gl_account_id ← HIST_TRAN_DTL_TABLE.ACID (VARCHAR2(11)) — ACID is Finacle's account ID (e.g., "0012345678901"), not a GL sub-head code (e.g., "10101") |
| Affected Table | `silver.gl.gl_transaction` |
| Affected Column(s) | `gl_account_id` |
| Confidence | 0.95 |

**Description:** In Finacle, `ACID` is the internal account identifier (CASA/Loan account number), which is completely different from the GL account code (`GL_SUB_HEAD_CODE`). Mapping `gl_account_id` from ACID means every GL transaction will have an account ID where a GL code is expected, making `JOIN silver.gl.gl_master ON gl_account_id` produce zero valid matches.

**Remediation:** Correct source to the GL sub-head code column. In Finacle, the GL account code on a transaction can typically be derived via `HIST_TRAN_DTL_TABLE → GL_SUB_HEAD_TABLE` through the transaction code. Document the derivation logic. For PRIME and RLS, the existing GLACCOUNTS.CODE / EQUATION_ACCT_CODE mappings appear correct.

**Estimated Effort:** Large
**Owner:** ETL

---

### Finding GL-TRANS-005 — Naming Convention Violations

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | Medium |
| Guideline Rule | `NAM-001` — snake_case, lowercase |
| Evidence | Mapping: `currency code` (space), `Account_number` (mixed case), `Agreement_Id` (mixed case), `Source_transaction_host_reference` (mixed case prefix) |
| Affected Table | `silver.gl.gl_transaction` |
| Affected Column(s) | `currency code`, `Account_number`, `Agreement_Id`, `Source_transaction_host_reference` |
| Confidence | 0.99 |

**Description:** Four columns violate NAM-001. The space in `currency code` will cause DDL syntax errors or force quoting everywhere.

**Remediation:** Rename: `currency code` → `currency_code`, `Account_number` → `account_bk`, `Agreement_Id` → `contract_bk`, `Source_transaction_host_reference` → `source_transaction_host_reference`.

**Estimated Effort:** Small
**Owner:** Data Modeling

---

### Finding GL-TRANS-006 — Missing DR/CR Explicit Indicator

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | Medium |
| Guideline Rule | Guideline 11 §2.1 "entity boundary and grain" |
| Evidence | Logical model: only `amount` with definition "Signed amount (DR=+, CR=-)" — no explicit posting_type_code or dr_cr_indicator |
| Affected Table | `silver.gl.gl_transaction` |
| Affected Column(s) | `amount` (missing companion) |
| Confidence | 0.88 |

**Description:** Signed amount is a valid convention for representing double-entry postings, but without an explicit `dr_cr_indicator` or `posting_type_code`, data consumers must know and rely on the sign convention. This is fragile across multi-source integrations where Finacle, PRIME, and RLS may use inconsistent sign conventions. An explicit debit/credit flag makes the posting semantic unambiguous.

**Remediation:** Add `dr_cr_indicator STRING NOT NULL` with canonical values 'DR' / 'CR' (sourced via `silver.reference.code_mapping` per SLV-004). Retain the signed amount for arithmetic convenience.

**Estimated Effort:** Small
**Owner:** Data Modeling

---

### 4.3 GL_RECONCILIATION

**Physical Table:** `silver_dev.gl.gl_reconciliation`
**Logical Definition:** Sub-ledger to GL reconciliation events
**Grain:** One row per reconciliation event
**Size:** 2.57 GB across 4 files — largest table in the subject area

#### 4.3.1 Industry Fitness

GL reconciliation tracks the match status between sub-ledger transactions and GL postings. This is a valid Silver entity for the Finance GL SA. However:
- `recon_status` (MATCHED/UNMATCHED/ADJUSTMENT) represents a **derived state** — it is computed by comparing two transaction streams, not directly sourced. This may violate SLV-006 (no derived values in Silver) if the status is computed during ETL rather than sourced from a reconciliation system.
- The entity references both `transaction_id (from GL)` and `transaction_id (from Txn Subject Area)` — a cross-subject-area reference to `silver.transaction`, which is acceptable as a foreign key reference but should be clearly documented.
- No partitioning on the largest table (2.57 GB) is a significant performance gap.

#### 4.3.2 Attribute-Level Review

| Attribute (Logical) | Logical Type | Mapping Status | Physical Source | Issues |
|---|---|---|---|---|
| `reconciliation_id` | STRING | Unmapped | No source | 🔴 Primary identifier has no source — must be generated or sourced |
| `transaction_id` | STRING | Mapped (FINACLE: TRAN_ID) | HIST_TRAN_DTL_TABLE.TRAN_ID | 🟡 FK to GL_TRANSACTIONS but transaction_id from GL_TRANSACTIONS may already be wrong (see GL-TRANS-004) |
| `source_system` | STRING | Unmapped | No source | 🔴 Critical for multi-source recon |
| `source_txn_id` | STRING | Unmapped | No source | 🔴 Unmapped |
| `recon_status` | STRING | Unmapped | No source | 🔴 Unmapped; if this is a computed/derived status, it violates SLV-006 |
| `source_transaction_host_reference` | STRING | Unmapped | No source | 🔴 Unmapped |
| `transaction_id__from_Txn_Subject_Area` | STRING | Mapped (PRIME: CTRANSACTIONS.SERNO) | PRIME only | 🟡 Double-underscore in name (NAM-001); cross-SA FK |
| `transaction_id__from_GL` | STRING | Unmapped | No source | 🔴 NAM-001 (double-underscore) |
| `Account_number` | STRING | Mapped (multi-source) | GENERAL_ACCT_MAST_TABLE.ACCT_NUM | 🟡 NAM-001 (mixed case); should be `account_bk` |
| `Agreement_Id` | STRING | Mapped (FINACLE, RLS) | FORACID / AGREEMENTID | 🟡 NAM-001 (mixed case); should be `contract_bk` |
| `gl_reconciliation_sk` | — | Missing | — | 🔴 SLV-001 |
| `gl_reconciliation_bk` | — | Missing | — | 🔴 SLV-002 |
| Metadata columns (11) | — | Missing | — | 🔴 SLV-008 |

#### 4.3.3 Metadata Completeness

All 11 mandatory metadata columns absent (same pattern as GL_MASTER).

---

### Finding GL-RECON-001 — Missing Surrogate and Business Key

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | `SLV-001`, `SLV-002` |
| Evidence | Logical model: no `gl_reconciliation_sk` or `gl_reconciliation_bk` |
| Affected Table | `silver.gl.gl_reconciliation` |
| Affected Column(s) | `gl_reconciliation_sk` (missing), `gl_reconciliation_bk` (missing) |
| Confidence | 0.98 |

**Description:** No SK/BK. `reconciliation_id` is the candidate BK but is itself unmapped. Without a stable identifier, the 2.57 GB reconciliation table cannot be deduplicated or incrementally loaded.

**Remediation:** Define `reconciliation_id` derivation logic (e.g., `MD5(source_system || '|' || transaction_id || '|' || source_txn_id)`). Add `gl_reconciliation_sk` and `gl_reconciliation_bk`.

**Estimated Effort:** Small
**Owner:** Data Modeling / ETL

---

### Finding GL-RECON-002 — `recon_status` May Be a Derived Value (SLV-006)

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | Medium |
| Guideline Rule | `SLV-006` — "3NF; no derived values/aggregations in Silver" |
| Evidence | Logical model: `recon_status` definition "MATCHED, UNMATCHED, ADJUSTMENT" — unmapped, no source system column |
| Affected Table | `silver.gl.gl_reconciliation` |
| Affected Column(s) | `recon_status` |
| Confidence | 0.80 |

**Description:** If `recon_status` is computed during Silver ETL by comparing GL transaction amounts to sub-ledger transaction amounts, it is a derived value and violates SLV-006. Reconciliation status should either: (a) be sourced directly from a reconciliation system that assigns status, or (b) be moved to a Gold reconciliation report.

**Remediation:** Clarify with data steward: (1) Is there a source system that publishes reconciliation status? If yes, map from source. (2) If status is computed, move to Gold layer. (3) If required in Silver for operational use, obtain steward approval and document in the Denormalization Register.

**Estimated Effort:** Small
**Owner:** Data Modeling / Governance

---

### Finding GL-RECON-003 — No Partitioning on 2.57 GB Table

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | Medium |
| Guideline Rule | Guideline 11 §1.4 "Partition all tables"; BCBS 239 Principle 2 (data accuracy and timeliness) |
| Evidence | Physical structures CSV: `partitionColumns: []`, `clusteringColumns: []`, sizeInBytes: 2,570,611,235, numFiles: 4 |
| Affected Table | `silver.gl.gl_reconciliation` |
| Affected Column(s) | Table-level (no partition) |
| Confidence | 0.99 |

**Description:** The reconciliation table at 2.57 GB with no partition or clustering columns will result in full-table scans for every incremental pipeline run and analytical query. This is the largest table in the GL subject area and will degrade performance at scale. Recommended partition column: `transaction_date` (or `ingested_date` once available).

**Remediation:** Add `PARTITIONED BY (transaction_date DATE)` or implement Databricks liquid clustering on `transaction_date` + `source_system`. Rebuild table with partition strategy.

**Estimated Effort:** Medium
**Owner:** DBA / ETL

---

### Finding GL-RECON-004 — Naming Convention Violations

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | Medium |
| Guideline Rule | `NAM-001` |
| Evidence | Mapping: `transaction_id__from_Txn_Subject_Area` (double-underscore), `transaction_id__from_GL` (double-underscore), `Account_number` (mixed case), `Agreement_Id` (mixed case) |
| Affected Table | `silver.gl.gl_reconciliation` |
| Affected Column(s) | 4 columns |
| Confidence | 0.99 |

**Description:** Double-underscore separators and mixed-case names violate NAM-001.

**Remediation:** Rename: `transaction_id__from_Txn_Subject_Area` → `transaction_sa_bk`, `transaction_id__from_GL` → `gl_transaction_bk`, `Account_number` → `account_bk`, `Agreement_Id` → `contract_bk`.

**Estimated Effort:** Small
**Owner:** Data Modeling

---

### 4.4 GL_ADJUSTMENTS

**Physical Table:** `silver_dev.gl.gl_adjustment`
**Logical Definition:** Manual and automated GL correction entries
**Grain:** One row per adjustment entry

#### 4.4.1 Industry Fitness

GL adjustments are manual or automated corrections to GL postings (e.g., fee errors, FX reclassification). This is a valid and necessary Silver entity. However, the entity lacks: an adjustment date/timestamp, an approval workflow reference (who approved the adjustment), and a reference to the original transaction being corrected. The `Whodidtheadjustment` field (violates NAM-001) only captures the user who made the adjustment, not the approver.

#### 4.4.2 Attribute-Level Review

| Attribute (Logical) | Logical Type | Mapping Status | Physical Source | Issues |
|---|---|---|---|---|
| `adjustment_id` | STRING | **WRONG** mapping | TBAADM.HIST_TRAN_DTL_TABLE.**GL_SUB_HEAD_CODE** (VARCHAR2(5)) | 🔴 CRITICAL: GL_SUB_HEAD_CODE is a GL account code, NOT an adjustment ID |
| `gl_account_id` | STRING | Mapped (multi-source) | HIST_TRAN_DTL_TABLE.ACID / GLACCOUNTS.CODE / EQUATION_ACCT_CODE | 🔴 Same ACID mismatch as GL_TRANSACTIONS; FINACLE source wrong |
| `amount` | STRING | Mapped (multi-source) | HIST_TRAN_DTL_TABLE.TRAN_AMT (NUMBER(20,4)) / GLLOGDAILYDETAILS.AMOUNT | 🔴 Type STRING must be DECIMAL(18,4) |
| `reason_code` | STRING | Mapped (PRIME only) | GLLOGDAILYDETAILS.REASONCODE | 🟡 Only PRIME mapped; FINACLE/RLS not mapped; raw source code not canonicalized |
| `Adjustment_mode` | STRING | Unmapped | No source | 🔴 NAM-001 (mixed case); should be `adjustment_mode_code` |
| `Whodidtheadjustment` | STRING | Mapped (FINACLE) | HIST_TRAN_DTL_TABLE.LCHG_USER_ID (VARCHAR2(15)) | 🔴 NAM-001 (no snake_case); should be `adjusted_by_user_id` |
| Metadata columns (11) | — | Missing | — | 🔴 SLV-008 |

---

### Finding GL-ADJ-001 — Critical: `adjustment_id` Mapped to GL Account Code

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | Guideline 11 §2.1 "entity grain and correctness" |
| Evidence | Mapping: `adjustment_id` ← TBAADM.HIST_TRAN_DTL_TABLE.GL_SUB_HEAD_CODE (VARCHAR2(5 CHAR)). GL_SUB_HEAD_CODE is the GL account sub-head code (e.g., "10101") — it is a dimension FK, not a unique adjustment identifier |
| Affected Table | `silver.gl.gl_adjustment` |
| Affected Column(s) | `adjustment_id` |
| Confidence | 0.97 |

**Description:** The primary identifier of GL_ADJUSTMENTS is mapped from `GL_SUB_HEAD_CODE` — a GL account code that repeats across every transaction for the same account. This mapping makes `adjustment_id` non-unique and semantically incorrect. Every adjustment for the same GL account would have the same "ID," making the table impossible to use as an adjustment register.

**Remediation:** Identify the correct unique adjustment identifier in Finacle. Likely candidates: `HIST_TRAN_DTL_TABLE.TRAN_ID` (transaction ID) combined with an adjustment type flag, or a dedicated adjustment/journal entry table. Consult Finacle DBA to confirm the source adjustment PK.

**Estimated Effort:** Medium
**Owner:** ETL / Data Modeling

---

### Finding GL-ADJ-002 — Missing Surrogate and Business Key

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | `SLV-001`, `SLV-002` |
| Evidence | Logical model: no `gl_adjustment_sk` or `gl_adjustment_bk` |
| Affected Table | `silver.gl.gl_adjustment` |
| Affected Column(s) | `gl_adjustment_sk` (missing), `gl_adjustment_bk` (missing) |
| Confidence | 0.98 |

**Description:** No SK/BK. Combined with the wrong `adjustment_id` mapping (GL-ADJ-001), this entity has no reliable unique identifier.

**Remediation:** After correcting the `adjustment_id` source, add `gl_adjustment_sk` (MD5 of correct BK) and `gl_adjustment_bk`.

**Estimated Effort:** Small
**Owner:** Data Modeling

---

### Finding GL-ADJ-003 — `amount` Typed as STRING

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | `SLV-010` — "Monetary amounts in smallest unit"; DQ Gate 2 "DECIMAL(18,4)" |
| Evidence | Logical model: `amount STRING`; source is NUMBER(20,4) from Finacle |
| Affected Table | `silver.gl.gl_adjustment` |
| Affected Column(s) | `amount` |
| Confidence | 0.99 |

**Description:** Adjustment amount typed as STRING despite an available numeric source. Monetary amounts must be DECIMAL(18,4) minimum, or BIGINT (fils for AED) per SLV-010.

**Remediation:** Change to DECIMAL(18,4). Add `amount_currency_code STRING` companion column. For AED: store in fils as BIGINT per SLV-010.

**Estimated Effort:** Small
**Owner:** Data Modeling

---

### Finding GL-ADJ-004 — Severe Naming Violations

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | Medium |
| Guideline Rule | `NAM-001` |
| Evidence | Logical model and mapping: `Adjustment_mode`, `Whodidtheadjustment` |
| Affected Table | `silver.gl.gl_adjustment` |
| Affected Column(s) | `Adjustment_mode`, `Whodidtheadjustment` |
| Confidence | 0.99 |

**Description:** `Whodidtheadjustment` is particularly egregious — it is a multi-word identifier with no separators and violates all naming standards. `Adjustment_mode` uses mixed case.

**Remediation:** `Adjustment_mode` → `adjustment_mode_code`; `Whodidtheadjustment` → `adjusted_by_user_id` (or `adjusted_by_employee_bk` if linked to the employee entity in `silver.org`).

**Estimated Effort:** Small
**Owner:** Data Modeling

---

### Finding GL-ADJ-005 — Missing Adjustment Date and Approval Reference

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | Medium |
| Guideline Rule | Guideline 11 §2.1 "industry fitness"; BCBS 239 Principle 2 |
| Evidence | Logical model: no `adjustment_date`, no `approved_by_user_id`, no `original_transaction_id` |
| Affected Table | `silver.gl.gl_adjustment` |
| Affected Column(s) | Missing attributes |
| Confidence | 0.90 |

**Description:** A GL adjustment register that tracks who made an adjustment but not when it was made (`adjustment_date`), who approved it (`approved_by_user_id`), or which original transaction it corrects (`original_transaction_id`) is insufficient for SOX/CBUAE audit trail requirements.

**Remediation:** Add: `adjustment_date TIMESTAMP NOT NULL`, `approved_by_user_id STRING`, `original_transaction_bk STRING` (FK to GL_TRANSACTIONS).

**Estimated Effort:** Small
**Owner:** Data Modeling

---

### 4.5 PRODUCT_GL_MAPPING

**Physical Table:** `silver_dev.gl.product_gl_mapping`
**Logical Definition:** Mapping table linking product/transaction codes to GL accounts and posting rules
**Grain:** One row per `(source_system, product_code, txn_type, gl_account_id)` combination (composite key, not explicitly defined)

#### 4.5.1 Industry Fitness

Product-to-GL posting rule mapping is a legitimate operational reference entity for a Finance GL SA — it defines which GL account should be debited or credited for each product/transaction type combination. In IFW, this corresponds to the "Booking Rules" pattern.

However, this entity has several design concerns:
- **No composite PK is defined.** The grain is implicit. Multiple rows for the same product/txn combination with different GL accounts is possible.
- **Cross-SA reference:** `product_code` references `silver.product` — the relationship must be documented with a FK.
- **`txn_type` and `txn_code` appear duplicated** — both mapped from `GLTRXNTYPES.TRXNTYPESERNO` in PRIME.
- The table is 4.2 MB (large enough to be a production reference table) — this entity has been implemented before being properly modeled.

#### 4.5.2 Attribute-Level Review

| Attribute (Logical) | Logical Type | Mapping Status | Physical Source | Issues |
|---|---|---|---|---|
| `source_system` | STRING | Unmapped | No source | 🔴 Unmapped; required for multi-source disambiguation |
| `product_code` | STRING | Mapped (multi-source) | GENERAL_ACCT_MAST_TABLE.SCHM_CODE / GLTRANSACTIONTYPES.PRODUCTSERNO | 🟡 Source columns are scheme code and product serial — canonical product code derivation not documented |
| `txn_type` | STRING | Mapped (FINACLE, PRIME) | TRANCODE_DESC (VARCHAR2(50)) / GLTRXNTYPES.TRXNTYPESERNO | 🟡 PRIME maps same source as txn_code — ambiguous; txn_type should be a canonical type, not source description |
| `txn_code` | STRING | Mapped (FINACLE, PRIME) | TRAN_CODE (VARCHAR2(5)) / GLTRXNTYPES.TRXNTYPESERNO | 🟡 PRIME maps same column as txn_type — duplicate mapping |
| `gl_account_id` | STRING | Mapped (multi-source) | ACID / GLACCOUNTS.CODE / EQUATION_ACCT_CODE | 🔴 Same ACID mismatch for FINACLE as other GL entities |
| `gl_posting_rule` | STRING | **WRONG** mapping | CUSTOM.C_HIST_FIN_TRAN_TABLE.**POSTING_USERID** (VARCHAR2(60)) | 🔴 CRITICAL: POSTING_USERID is a user ID, not a DR/CR posting rule |
| `vat_applicable` | STRING | Unmapped | No source | 🔴 Unmapped; typo in description ("FASLE" instead of "FALSE"); should be BOOLEAN |
| `central_bank_code` | STRING | **WRONG** mapping | CRMUSER.ACCOUNTS.**IS_SWIFT_CODE_OF_BANK** (NVARCHAR2(3)) / TCTDBS.BANKS.BANKID | 🔴 CRITICAL: IS_SWIFT_CODE_OF_BANK is a SWIFT BIC code, not a CBUAE reporting code |
| Metadata columns (11) | — | Missing | — | 🔴 SLV-008 |

---

### Finding GL-PGM-001 — Critical: `gl_posting_rule` Mapped to User ID

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | Guideline 11 §2.1 "entity grain and correctness" |
| Evidence | Mapping: `gl_posting_rule` ← CUSTOM.C_HIST_FIN_TRAN_TABLE.POSTING_USERID (VARCHAR2(60)). POSTING_USERID is clearly a user identifier (person who posted), not a DR/CR posting rule |
| Affected Table | `silver.gl.product_gl_mapping` |
| Affected Column(s) | `gl_posting_rule` |
| Confidence | 0.96 |

**Description:** The GL posting rule (expected values: 'DR' or 'CR' to indicate debit or credit posting direction) is mapped from a user ID field. This will populate posting rules with usernames/IDs instead of DR/CR codes, completely corrupting the product-to-GL posting rule reference. Any downstream Gold posting logic that relies on `gl_posting_rule` will produce incorrect journal entries.

**Remediation:** Identify the correct source column for DR/CR posting direction in FINACLE's transaction code table (likely `TBAADM.TRAN_CODE_TABLE` or `CUSTOM.C_HIST_FIN_TRAN_TABLE.DR_CR_FLG` or equivalent). Remap accordingly and validate against the `silver.reference.code_mapping` for canonical DR/CR values.

**Estimated Effort:** Medium
**Owner:** ETL / Data Modeling

---

### Finding GL-PGM-002 — Critical: `central_bank_code` Mapped to SWIFT/Bank ID

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | Guideline 11 §2.1 "entity grain and correctness" |
| Evidence | Mapping: `central_bank_code` ← CRMUSER.ACCOUNTS.IS_SWIFT_CODE_OF_BANK (NVARCHAR2(3)) / TCTDBS.BANKS.BANKID. SWIFT codes (BIC) are bank routing identifiers; CBUAE regulatory reporting codes (e.g., B201, T401) are CBUAE-specific classification codes |
| Affected Table | `silver.gl.product_gl_mapping` |
| Affected Column(s) | `central_bank_code` |
| Confidence | 0.94 |

**Description:** CBUAE central bank reporting codes are product-level regulatory codes used for CBUAE Form reporting (e.g., CBUAE Retail Loans Return). These are typically defined in a separate CBUAE regulatory lookup table, not in ACCOUNTS.IS_SWIFT_CODE_OF_BANK (which stores a flag indicating whether a SWIFT code exists on an account) or BANKS.BANKID (which is an internal bank identifier). The mapping will populate `central_bank_code` with incorrect values.

**Remediation:** Obtain the CBUAE reporting code mapping table from the source systems. In Finacle, CBUAE codes are typically in a regulatory reporting configuration table. Consult the regulatory reporting team for the correct source.

**Estimated Effort:** Medium
**Owner:** ETL / Regulatory Reporting

---

### Finding GL-PGM-003 — Missing PK / Composite Key Definition

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | Medium |
| Guideline Rule | `SLV-001`, `SLV-002`; guideline 11 §2.3 "entity grain" |
| Evidence | Logical model: no composite key or SK/BK defined for PRODUCT_GL_MAPPING |
| Affected Table | `silver.gl.product_gl_mapping` |
| Affected Column(s) | Missing `product_gl_mapping_sk`, `product_gl_mapping_bk` |
| Confidence | 0.97 |

**Description:** No PK is defined. The natural grain is `(source_system, product_code, txn_type, gl_account_id)` — a composite business key. Without a defined PK, deduplication and incremental loads cannot be reliably implemented. The 4.2 MB size suggests data is already loaded without proper key management.

**Remediation:** Define composite BK: `MD5(CONCAT_WS('|', source_system, product_code, txn_type, gl_account_id))` → `product_gl_mapping_sk`. Add `product_gl_mapping_bk`.

**Estimated Effort:** Small
**Owner:** Data Modeling

---

### Finding GL-PGM-004 — `vat_applicable` Typed as STRING with Typo in Description

| Field | Value |
|---|---|
| Priority | P2 |
| Criticality | Low |
| Guideline Rule | `NAM-001` (is_/has_ prefix convention for BOOLEAN) |
| Evidence | Logical model: `vat_applicable STRING`; description "Example, FASLE, TRUE, FALSE" — typo "FASLE" confirms lack of review |
| Affected Table | `silver.gl.product_gl_mapping` |
| Affected Column(s) | `vat_applicable` |
| Confidence | 0.95 |

**Description:** `vat_applicable` is a Boolean indicator (yes/no VAT applies). It must be renamed to `is_vat_applicable` (BOOLEAN) per naming conventions. The typo "FASLE" in the description indicates this field has not been reviewed for accuracy.

**Remediation:** Rename to `is_vat_applicable BOOLEAN NOT NULL`. Add source mapping. Correct description typo.

**Estimated Effort:** Small
**Owner:** Data Modeling

---

### 4.6 VAT_RULES

**Physical Table:** `silver_dev.gl.vat_rules`
**Logical Definition:** VAT rate rules per GL account
**Grain:** One row per GL account VAT rule

#### 4.6.1 Industry Fitness

VAT_RULES defines UAE VAT (5%) applicability and rates per GL account. This is reference/master data — it does not belong in `silver.gl`. VAT rate reference data is a domain-agnostic regulatory reference table that should live in `silver.reference` and be consumed by multiple subject areas (GL, product, transaction, etc.). Placing it in `silver.gl` violates the subject-area isolation principle and forces all non-GL consumers to read from the GL schema.

Additionally, `vat_rate` is typed as STRING (should be DECIMAL(5,4)), and the mapping sources are questionable.

#### 4.6.2 Attribute-Level Review

| Attribute (Logical) | Logical Type | Mapping Status | Physical Source | Issues |
|---|---|---|---|---|
| `gl_account_id` | STRING | Mapped (multi-source) | GL_SUB_HEAD_TABLE.GL_CODE / GLACCOUNTS.CODE / GL_BALANCE.EQUATION_ACCT_CODE | 🟡 Both `gl_account_id` and `vat_gl_account` mapped from same GL_CODE column in FINACLE — how are two different accounts derived from one column? |
| `vat_gl_account` | STRING | Mapped (FINACLE, PRIME) | GL_SUB_HEAD_TABLE.GL_CODE / GLACCOUNTS.CODE | 🔴 Same source column as `gl_account_id` — mapping cannot be correct for both |
| `vat_rate` | STRING | Mapped (RLS only) | FIN_LEAM.LEA_AGREEMENT_DTL.VAT_PRIN_RATE | 🔴 This is a loan-level VAT rate on principal — not a product-level VAT rule rate; wrong source |
| `vat_description` | STRING | Mapped (PRIME only) | TCTDBS.GLACCOUNTS.DESCRIPTION | 🟡 Account description ≠ VAT description; likely wrong semantic |
| Metadata columns | — | Missing | — | 🔴 SLV-008 |

---

### Finding GL-VAT-001 — Entity Misplaced: Should Be in `silver.reference`

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | Guideline 11 §2.2 "Subject-Area Isolation"; `sa/subject-areas.md` — reference data belongs in `silver.reference` |
| Evidence | `sa/subject-areas.md`: SA #1 Reference Data: "Master Lookups: ISO Currencies, Country Codes, FX Rates, Commodity Prices (Gold/Silver), Channel Master, and Industry Codes" — VAT rules are UAE regulatory reference data |
| Affected Table | `silver_dev.gl.vat_rules` |
| Affected Column(s) | Entire table |
| Confidence | 0.92 |

**Description:** UAE VAT rates are UAE FTA regulatory reference data — they should be registered in `silver.reference.vat_rules` for reuse by all subject areas (Transaction, Product, GL). Placing them in `silver.gl` means any non-GL pipeline needing VAT rates must violate the cross-SA isolation principle by reading from the GL schema.

**Remediation:** Move `vat_rules` to `silver.reference` schema. Update all pipeline references. Register in Reference Data SA governance.

**Estimated Effort:** Medium
**Owner:** Data Modeling / ETL

---

### Finding GL-VAT-002 — `gl_account_id` and `vat_gl_account` Share the Same Source Column

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | Guideline 11 §2.1 "entity correctness" |
| Evidence | Mapping: both `gl_account_id` and `vat_gl_account` ← TBAADM.GL_SUB_HEAD_TABLE.GL_CODE (FINACLE) and TCTDBS.GLACCOUNTS.CODE (PRIME) |
| Affected Table | `silver_dev.gl.vat_rules` |
| Affected Column(s) | `gl_account_id`, `vat_gl_account` |
| Confidence | 0.96 |

**Description:** Both the primary GL account (`gl_account_id`, e.g., a revenue account) and the VAT liability GL account (`vat_gl_account`, e.g., a VAT payable account) are mapped from the same source column. These must be two different GL account codes — the business account and the VAT counterpart account. The mapping cannot correctly differentiate them from a single source column.

**Remediation:** Identify the correct source columns for both accounts. VAT rules are typically configured in a VAT configuration table (not the GL account master itself). Consult Finacle's Tax/VAT configuration module for the correct source tables.

**Estimated Effort:** Medium
**Owner:** ETL / Regulatory Reporting

---

### Finding GL-VAT-003 — `vat_rate` Sourced from Loan VAT Principal Rate

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | Medium |
| Guideline Rule | Guideline 11 §2.1 "entity correctness" |
| Evidence | Mapping: `vat_rate` ← FIN_LEAM.LEA_AGREEMENT_DTL.VAT_PRIN_RATE. VAT_PRIN_RATE is a loan agreement-level VAT rate on principal — not a product/GL-level VAT rate |
| Affected Table | `silver_dev.gl.vat_rules` |
| Affected Column(s) | `vat_rate` |
| Confidence | 0.88 |

**Description:** UAE VAT on financial services is a flat 5% (or exempt in specific cases). The rate stored in a loan agreement's VAT_PRIN_RATE field is a transaction-specific rate, not the authoritative product-level VAT rule. Additionally, `vat_rate` is typed as STRING — it should be DECIMAL(5,4).

**Remediation:** Source VAT rate from the authoritative UAE FTA rate configuration (standard 5% = 0.0500). If source-system configuration is needed, identify the VAT configuration table in FINACLE/RLS. Change type to DECIMAL(5,4).

**Estimated Effort:** Small
**Owner:** ETL / Regulatory Reporting

---

### 4.7 CENTRAL_BANK_CODES

**Physical Table:** `silver_dev.gl.central_bank_code`
**Logical Definition:** CBUAE reporting category codes
**Grain:** One row per CBUAE reporting code

#### 4.7.1 Industry Fitness

CBUAE Central Bank reporting codes are UAE regulatory classification codes used to categorize banking exposures for CBUAE statistical returns (e.g., retail loans return, deposits return). This is pure reference data that belongs in `silver.reference`, not `silver.gl`. It is used across lending, product, GL, and payments domains.

#### 4.7.2 Attribute-Level Review

| Attribute (Logical) | Logical Type | Mapping Status | Physical Source | Issues |
|---|---|---|---|---|
| `central_bank_code` | STRING | **WRONG** mapping | CRMUSER.ACCOUNTS.IS_SWIFT_CODE_OF_BANK / TCTDBS.BANKS.BANKID | 🔴 CRITICAL: IS_SWIFT_CODE_OF_BANK is a Boolean/SWIFT indicator on customer accounts; BANKID is an internal bank serial. Neither is a CBUAE reporting code |
| `description` | STRING | Mapped (FINACLE, PRIME) | CRMUSER.ACCOUNTS.CUST_SWIFT_CODE_DESC / TCTDBS.BANKS.NAME | 🔴 SWIFT code description / Bank name ≠ CBUAE reporting category description |
| `reporting_category` | STRING | Unmapped | No source | 🔴 Unmapped |
| Metadata columns | — | Missing | — | 🔴 SLV-008 |

---

### Finding GL-CBC-001 — Critical: All Three Mapped Attributes Use Wrong Source Columns

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | Guideline 11 §2.1 "entity correctness" |
| Evidence | central_bank_code ← IS_SWIFT_CODE_OF_BANK (NVARCHAR2(3), flag/SWIFT code); description ← CUST_SWIFT_CODE_DESC (SWIFT description) / BANKS.NAME (bank name). None of these are CBUAE regulatory codes |
| Affected Table | `silver_dev.gl.central_bank_code` |
| Affected Column(s) | `central_bank_code`, `description` |
| Confidence | 0.95 |

**Description:** The entire CENTRAL_BANK_CODES table is mapped from SWIFT/bank identifier source columns rather than from CBUAE regulatory reporting code tables. CBUAE codes (e.g., B201 = SME Conventional Overdraft, T401 = Trade LC) are configuration data typically maintained by the Regulatory Reporting team, not from customer account SWIFT fields. This mapping will populate the table with bank SWIFT identifiers rather than CBUAE reporting categories.

**Remediation:** (1) Move entity to `silver.reference`. (2) Identify the authoritative source for CBUAE reporting codes — typically a regulatory configuration table or manual reference maintained by the CBUAE Regulatory Reporting team. (3) Remap from correct source. (4) If no automated source exists, implement as a manually-loaded reference table with change management governance.

**Estimated Effort:** Medium
**Owner:** ETL / Regulatory Reporting / Governance

---

### Finding GL-CBC-002 — Entity Misplaced: Should Be in `silver.reference`

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | Guideline 11 §2.2 "Subject-Area Isolation"; `sa/subject-areas.md` |
| Evidence | CBUAE reporting codes are used by Lending, Transaction, Trade Finance, and Payments SAs — not just GL |
| Affected Table | `silver_dev.gl.central_bank_code` |
| Affected Column(s) | Entire table |
| Confidence | 0.92 |

**Description:** CBUAE codes are cross-domain regulatory reference data. All subject areas that report to CBUAE (Lending, Deposits, Payments, GL) need access to this lookup. Placing it in `silver.gl` forces cross-SA reads, violating isolation.

**Remediation:** Move to `silver.reference.central_bank_code`. Update all referencing pipelines.

**Estimated Effort:** Small
**Owner:** Data Modeling

---

## 4.8 Missing Critical Entities

### 4.8.1 GL_PERIOD / Fiscal Calendar

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Impact | Without a fiscal period entity, period-end financial statements (Balance Sheet, P&L) cannot be produced. Trial balances require a period cut-off reference. |
| Rationale | `sa/subject-areas.md` explicitly lists "Financial Period definitions" as part of Finance GL. BIAN SD-186 requires an Accounting Period component. |
| Effort | Medium |

**Required Attributes:**
- `fiscal_period_sk` (MD5 surrogate)
- `fiscal_period_bk` (e.g., '2025-Q4', '2025-12')
- `fiscal_year` INTEGER
- `fiscal_quarter` INTEGER (1-4)
- `fiscal_month` INTEGER (1-12)
- `period_start_date` DATE NOT NULL
- `period_end_date` DATE NOT NULL
- `period_status_code` STRING (OPEN/CLOSED/ADJUSTMENT)
- `is_current_period` BOOLEAN
- Standard audit/SCD columns

---

### 4.8.2 GL_BALANCE / Trial Balance

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Impact | Without period-end balances, the system cannot produce a Trial Balance, Balance Sheet, or P&L — core deliverables of the Finance GL SA. The `gl_reconciliation` table is 2.57 GB of transactional data but has no period-balance summary entity. |
| Rationale | `sa/subject-areas.md` explicitly lists "Trial Balances" as part of Finance GL. IFW requires an Account Balance entity. |
| Effort | Large |

**Required Attributes:**
- `gl_balance_sk` (MD5 surrogate)
- `gl_balance_bk` (composite: gl_account_id + fiscal_period_bk + currency_code)
- `gl_master_sk` FK
- `fiscal_period_sk` FK
- `opening_balance_amount` DECIMAL(18,4)
- `closing_balance_amount` DECIMAL(18,4)
- `debit_amount` DECIMAL(18,4)
- `credit_amount` DECIMAL(18,4)
- `currency_code` STRING
- `source_system_code` STRING
- Standard audit/SCD columns

---

### 4.8.3 GL_ACCOUNT_HIERARCHY

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | Medium |
| Impact | `gl_parent_account_id` exists in the GL_MASTER logical model but a recursive hierarchy in a single table is insufficient for multi-level CoA roll-ups required for Balance Sheet / P&L line items. |
| Rationale | IFW CoA pattern requires a bridge/hierarchy table for arbitrary-depth account trees. BCBS 239 Principle 6 (adaptability) requires risk data aggregation across CoA hierarchy. |
| Effort | Medium |

**Required Attributes:**
- `hierarchy_node_sk` (MD5 surrogate)
- `gl_account_sk` FK (leaf or parent node)
- `parent_account_sk` FK (nullable for root)
- `hierarchy_level` INTEGER
- `hierarchy_path` STRING (e.g., '/Assets/CurrentAssets/CashEquivalents/')
- `is_leaf` BOOLEAN
- Standard audit/SCD columns

---

## 5. Denormalization Register

| # | Entity | Denormalized Attribute | Canonical Location | Classification | Justification | Decision |
|---|---|---|---|---|---|---|
| 1 | GL_MASTER | `product_type` | `silver.product` via PRODUCT_GL_MAPPING | ❌ Unnecessary | No documented justification for including product classification in the CoA master | **Remove**: Replace with FK to `silver.product` |
| 2 | GL_TRANSACTIONS | `Account_number` (account_bk) | `silver.account.account.account_bk` | 🟡 Acceptable | Provides lineage from GL posting back to originating account; operational need | **Retain as FK reference** with explicit documentation |
| 3 | GL_TRANSACTIONS | `Agreement_Id` (contract_bk) | `silver.contract.contract.contract_bk` | 🟡 Acceptable | Provides lineage from GL posting to originating contract; needed for IFRS 9 | **Retain as FK reference** with explicit documentation |
| 4 | GL_RECONCILIATION | `Account_number` | `silver.account.account.account_bk` | 🟡 Acceptable | Reconciliation context; same justification as GL_TRANSACTIONS | **Retain as FK reference** |
| 5 | GL_RECONCILIATION | `Agreement_Id` | `silver.contract.contract.contract_bk` | 🟡 Acceptable | Same as GL_TRANSACTIONS | **Retain as FK reference** |
| 6 | PRODUCT_GL_MAPPING | `central_bank_code` | `silver.reference.central_bank_code` | ❌ Unnecessary | CENTRAL_BANK_CODES should be a standalone reference entity; no need to embed the code in a mapping table (a FK is sufficient) | **Replace with FK** once CENTRAL_BANK_CODES is in `silver.reference` |
| 7 | PRODUCT_GL_MAPPING | `vat_applicable` | `silver.reference.vat_rules` | ❌ Unnecessary | VAT applicability per product is already captured in VAT_RULES — duplicate | **Remove**: Use JOIN to `silver.reference.vat_rules` |

**Summary:** 2 denormalizations are necessary (account/contract BK lineage references), 5 are unnecessary or should be replaced by FK references. No accepted necessary denormalizations without documented justification.

---

## 6. Guideline Compliance Summary

| Rule | Description | Status | Entities Affected | Findings |
|---|---|---|---|---|
| **SLV-001** | `<entity>_sk` MD5 surrogate on every entity | 🔴 Fail | All 7 entities | GL-MASTER-001, GL-TRANS-001, GL-RECON-001, GL-ADJ-002, GL-PGM-003, GL-VAT, GL-CBC |
| **SLV-002** | `<entity>_bk` business key on every entity | 🔴 Fail | All 7 entities | Same as SLV-001 |
| **SLV-003** | SCD-2 default (`effective_from`, `effective_to`, `is_current`) | 🔴 Fail | All 7 entities | GL-MASTER-003 and equivalent across all entities |
| **SLV-004** | No source-system codes; use `silver.reference.code_mapping` | 🔴 Fail | GL_MASTER, GL_ADJUSTMENTS, GL_TRANSACTIONS | GL-MASTER-004; reason_code unmapped/uncanonicalized |
| **SLV-005** | DQ-gated writes; quarantine table | 🔴 No Evidence | All entities | No DQ gate logic visible in mapping; referential integrity not enforced (GL-MASTER-002) |
| **SLV-006** | 3NF; no derived values | 🟡 Partial | GL_RECONCILIATION | GL-RECON-002 (recon_status possibly derived) |
| **SLV-007** | No pre-computed metrics | 🟢 Pass | All entities | No aggregations detected in mapping |
| **SLV-008** | All metadata columns present | 🔴 Fail | All 7 entities | GL-MASTER-003 and equivalent across all entities — all 11 metadata columns absent |
| **SLV-009** | All timestamps UTC | 🟡 Unknown | All entities | Dates stored as STRING in logical model; no UTC conversion logic in mapping |
| **SLV-010** | Monetary amounts in smallest unit (fils for AED) | 🔴 Fail | GL_TRANSACTIONS, GL_ADJUSTMENTS | GL-TRANS-003, GL-ADJ-003 — amounts typed as STRING |
| **NAM-001** | snake_case, lowercase, no reserved words | 🔴 Fail | All 7 entities | GL-MASTER-005, GL-TRANS-005, GL-RECON-004, GL-ADJ-004, GL-PGM-004 — multiple violations |
| **NAM-003** | Table naming: `<entity>` for Silver | 🟢 Pass | All 7 physical tables | Physical tables use singular snake_case correctly |
| **Catalog Registration** | All tables must have descriptions in Unity Catalog | 🔴 Fail | All 7 tables | All table descriptions are `null` in physical structures CSV |
| **Data Catalog** | All tables registered in GDGC/Purview with steward and lineage | 🔴 Unknown | All 7 tables | No evidence of catalog registration |
| **Cross-SA Scoping** | VAT_RULES and CENTRAL_BANK_CODES belong in `silver.reference` | 🔴 Fail | VAT_RULES, CENTRAL_BANK_CODES | GL-VAT-001, GL-CBC-002 |
| **Missing Entities** | GL_PERIOD, GL_BALANCE, GL_ACCOUNT_HIERARCHY absent | 🔴 Fail | Subject area | Section 4.8 |

### Compliance Score

| Category | Passing | Total | Score |
|---|---|---|---|
| Structural Rules (SLV-001–010) | 1 | 10 | 10% |
| Naming Rules (NAM-001, NAM-003) | 1 | 2 | 50% |
| Mapping Completeness | 0 | 7 entities | 0% (no entity is fully mapped) |
| Entity Completeness | 7 | 10 (incl. missing) | 70% |
| **Overall** | **~9/31 checks passing** | — | **~29%** |

---

## 7. Remediation Plan

### 7.1 P0 — Immediate (Before Any Production Promotion)

| # | Action | Entity/Scope | Owner | Effort | Dependency |
|---|---|---|---|---|---|
| P0-1 | Add `<entity>_sk` and `<entity>_bk` to all 7 entities with correct MD5 derivation | All entities | Data Modeling | Large | Correct BK definition per entity |
| P0-2 | Fix `adjustment_id` mapping — identify correct source for unique adjustment ID in FINACLE | GL_ADJUSTMENTS | ETL | Medium | Finacle DBA consultation |
| P0-3 | Fix `gl_posting_rule` mapping — identify DR/CR rule column in FINACLE transaction code table | PRODUCT_GL_MAPPING | ETL | Medium | Finacle DBA consultation |
| P0-4 | Fix `central_bank_code` mapping — identify CBUAE reporting code source tables | PRODUCT_GL_MAPPING, CENTRAL_BANK_CODES | ETL / Reg Reporting | Medium | CBUAE code register |
| P0-5 | Fix `transaction_date` / `value_date` inversion — swap source column mappings | GL_TRANSACTIONS | ETL | Small | None |
| P0-6 | Correct `amount` type from STRING to DECIMAL(18,4) on GL_TRANSACTIONS and GL_ADJUSTMENTS | GL_TRANSACTIONS, GL_ADJUSTMENTS | Data Modeling / ETL | Medium | None |
| P0-7 | Fix `gl_account_id` FK mismatch — establish canonical GL account code and transformation logic across all source systems | All GL entities | ETL / Data Modeling | Large | Cross-source analysis |
| P0-8 | Add all 11 mandatory SCD-2 and audit columns to all 7 entities | All entities | Data Modeling / DBA | Large | Approved DDL |
| P0-9 | Define and implement GL_PERIOD entity | New entity | Data Modeling | Medium | Finacle fiscal calendar source |
| P0-10 | Define and implement GL_BALANCE (Trial Balance) entity | New entity | Data Modeling | Large | GL_PERIOD + GL_MASTER |
| P0-11 | Move VAT_RULES to `silver.reference` schema | VAT_RULES | Data Modeling / ETL | Medium | Reference SA data steward |
| P0-12 | Move CENTRAL_BANK_CODES to `silver.reference` schema | CENTRAL_BANK_CODES | Data Modeling / ETL | Medium | Reference SA data steward |
| P0-13 | Correct `vat_gl_account` / `gl_account_id` dual-mapping from same source column in VAT_RULES | VAT_RULES | ETL | Medium | VAT config table identification |
| P0-14 | Add table descriptions to all 7 Unity Catalog entries | All entities | Governance | Small | Business definitions |

### 7.2 P1 — Next Sprint

| # | Action | Entity/Scope | Owner | Effort |
|---|---|---|---|---|
| P1-1 | Fix all NAM-001 naming violations (spaces, PascalCase, mixed case) in logical model | All entities | Data Modeling | Medium |
| P1-2 | Canonicalize `gl_category` via `silver.reference.code_mapping` (SLV-004) | GL_MASTER | ETL / Governance | Small |
| P1-3 | Add explicit `dr_cr_indicator` to GL_TRANSACTIONS | GL_TRANSACTIONS | Data Modeling / ETL | Small |
| P1-4 | Change `is_reconcilable` from STRING to BOOLEAN; add source mapping | GL_MASTER | Data Modeling | Small |
| P1-5 | Add `adjustment_date`, `approved_by_user_id`, `original_transaction_bk` to GL_ADJUSTMENTS | GL_ADJUSTMENTS | Data Modeling | Small |
| P1-6 | Rename `Adjustment_mode` → `adjustment_mode_code`; `Whodidtheadjustment` → `adjusted_by_user_id` | GL_ADJUSTMENTS | Data Modeling | Small |
| P1-7 | Implement `reason_code` canonicalization for GL_ADJUSTMENTS via `silver.reference.code_mapping` | GL_ADJUSTMENTS | ETL / Governance | Small |
| P1-8 | Resolve `reconciliation_id` — define derivation logic; add source mapping | GL_RECONCILIATION | ETL / Data Modeling | Medium |
| P1-9 | Map `recon_status` from correct source; if computed, move to Gold | GL_RECONCILIATION | Data Modeling / Governance | Small |
| P1-10 | Add liquid clustering on `gl_reconciliation` (transaction_date + source_system) | GL_RECONCILIATION | DBA | Medium |
| P1-11 | Define and implement GL_ACCOUNT_HIERARCHY entity | New entity | Data Modeling | Medium |
| P1-12 | Correct `vat_rate` source in VAT_RULES to authoritative VAT config table | VAT_RULES | ETL / Reg Reporting | Small |
| P1-13 | Remove `product_type` from GL_MASTER; replace with documented FK reference | GL_MASTER | Data Modeling | Small |
| P1-14 | Fix `gl_parent_Account_id` type (CHAR(18) → STRING) and mapping | GL_MASTER | Data Modeling / ETL | Small |
| P1-15 | Document sign convention (DR=+, CR=−) in data dictionary for GL_TRANSACTIONS | GL_TRANSACTIONS | Governance | Small |
| P1-16 | Implement DQ gate for all 7 entities with quarantine routing | All entities | ETL | Large |
| P1-17 | Register all tables in GDGC/Purview with data steward, classification, and lineage | All entities | Governance | Medium |
| P1-18 | Map `source_system` attribute in GL_TRANSACTIONS, GL_RECONCILIATION, GL_ADJUSTMENTS | 3 entities | ETL | Small |

### 7.3 P2 — Backlog

| # | Action | Entity/Scope | Owner | Effort |
|---|---|---|---|---|
| P2-1 | Remove `vat_applicable` from PRODUCT_GL_MAPPING; use JOIN to `silver.reference.vat_rules` | PRODUCT_GL_MAPPING | Data Modeling | Small |
| P2-2 | Replace `central_bank_code` embedded column in PRODUCT_GL_MAPPING with FK | PRODUCT_GL_MAPPING | Data Modeling | Small |
| P2-3 | Define composite PK for PRODUCT_GL_MAPPING; add composite BK documentation | PRODUCT_GL_MAPPING | Data Modeling | Small |
| P2-4 | Implement UTC timestamp conversion for all date/timestamp fields across all entities | All entities | ETL | Medium |
| P2-5 | Develop cross-source GL account code reconciliation framework (FINACLE 5-char vs PRIME/RLS 13-char) | All entities | ETL / Data Modeling | Large |
| P2-6 | Rename `vat_applicable` → `is_vat_applicable BOOLEAN` | PRODUCT_GL_MAPPING | Data Modeling | Small |
| P2-7 | Map `txn_type` and `txn_code` to distinct source columns in PRODUCT_GL_MAPPING (currently same PRIME source) | PRODUCT_GL_MAPPING | ETL | Small |
| P2-8 | Add `journal_entry_id` to GL_TRANSACTIONS to group related debit/credit legs | GL_TRANSACTIONS | Data Modeling | Medium |
| P2-9 | Perform full mapping coverage expansion — map all currently unmapped attributes (≈71% of total) | All entities | ETL / Data Mapping | Large |

### 7.4 Proposed Delivery Schedule

| Sprint | Items | Target Completion |
|---|---|---|
| Sprint 1 (Weeks 1–2) | P0-1 through P0-8 (keys, types, audit columns, critical mapping fixes) | T+2 weeks |
| Sprint 2 (Weeks 3–4) | P0-9 through P0-14 (new entities, entity moves, catalog descriptions) | T+4 weeks |
| Sprint 3 (Weeks 5–6) | P1-1 through P1-9 (naming, canonicalization, recon improvements) | T+6 weeks |
| Sprint 4 (Weeks 7–8) | P1-10 through P1-18 (performance, DQ gate, catalog registration) | T+8 weeks |
| Sprint 5+ | P2 backlog | T+12 weeks |

---

## 8. Appendix

### Appendix A: Mapping Summary by Entity

| Entity | Attributes in Logical Model | Attributes in Mapping Document | Confirmed Mapped | Confirmed Unmapped | Confirmed Wrong | Mapping Coverage |
|---|---|---|---|---|---|---|
| GL_MASTER | 15 | 5 | 3 (gl_account_id, gl_account_name, gl_category) | 12 | 0 | 20% |
| GL_TRANSACTIONS | 18 | 11 | 9 | 9 | 2 (transaction_date, value_date inverted) | 50% (with errors) |
| GL_RECONCILIATION | 16 | 10 | 4 | 12 | 0 confirmed | 25% |
| GL_ADJUSTMENTS | 15 | 6 | 4 | 11 | 1 (adjustment_id) | 27% (with error) |
| PRODUCT_GL_MAPPING | 17 | 8 | 6 | 11 | 2 (gl_posting_rule, central_bank_code) | 35% (with errors) |
| VAT_RULES | 13 | 4 | 3 | 10 | 1 suspected (vat_rate source, gl_account_id = vat_gl_account) | 23% |
| CENTRAL_BANK_CODES | 9 | 3 | 0 (all wrong) | 7 | 2 confirmed (central_bank_code, description) | 0% correct |
| **TOTAL** | **103** | **47** | **~29** | **~72** | **8 confirmed** | **~28%** |

### Appendix B: Guideline Citations

| Rule Code | Quoted Rule Text | Source Document |
|---|---|---|
| SLV-001 | Every Silver entity must have `<entity>_sk` (MD5 deterministic surrogate key) | guidelines/11-modeling-dos-and-donts.md §2.4 "Surrogate Keys — Deterministic, Not Auto-Increment" |
| SLV-002 | Every Silver entity must have `<entity>_bk` (business/natural key) | guidelines/11-modeling-dos-and-donts.md §2.4 |
| SLV-003 | SCD-2 default: `effective_from`, `effective_to`, `is_current` | guidelines/11-modeling-dos-and-donts.md §2.3 "SCD Type 2 History" |
| SLV-004 | Resolve all source-system codes to canonical platform values using `silver.reference.code_mapping` | guidelines/11-modeling-dos-and-donts.md §2.5 "Business Key and Source Code Canonicalization" |
| SLV-005 | "Failed records routed to quarantine / error tables rather than discarded" | guidelines/01-data-quality-controls.md §3.2 "Enforcement" |
| SLV-006 | "Store only atomic, un-aggregated facts in Silver" | guidelines/11-modeling-dos-and-donts.md §2.6 "No Aggregations or Derived Metrics in Silver" |
| SLV-007 | "Prohibited in Silver; reserved for the Gold layer" | guidelines/01-data-quality-controls.md §4 "Aggregations/KPIs" |
| SLV-008 | All Silver and Gold tables must carry mandatory audit columns | guidelines/01-data-quality-controls.md §5 "Mandatory Audit Columns" and guidelines/11-modeling-dos-and-donts.md §2.7 "Mandatory Silver Technical Audit Columns" |
| SLV-009 | "All timestamps stored in UTC" | guidelines/01-data-quality-controls.md §4 "Time Zone Normalisation" |
| SLV-010 | "All monetary columns must use DECIMAL(18,4)" | guidelines/01-data-quality-controls.md §2 Gate 2 "Standardisation — Currencies" |
| NAM-001 | "Use snake_case for all identifiers; lowercase only; no abbreviations unless universally understood" | guidelines/06-naming-conventions.md §"General Rules" |
| NAM-003 | "Silver entity: `<entity>`" | guidelines/06-naming-conventions.md §"Table Naming" |
| NAM-005 | "Never use SQL reserved words as identifiers" | guidelines/06-naming-conventions.md §"General Rules" |

### Appendix C: Industry References

| Reference | Relevance |
|---|---|
| **BIAN v11 SD-186 — General Ledger Management** | Defines CoA, Journal Entry, Period Balance, and Reconciliation as required GL components. Confirms GL_PERIOD and GL_BALANCE are mandatory. |
| **IFW (IBM Industry Framework for Banking) — Financial Accounting domain** | Chart of Accounts hierarchy, Account Balance, Accounting Period patterns. GL_ACCOUNT_HIERARCHY and GL_BALANCE required. |
| **IFRS 9** | Period-end ECL staging requires accurate GL period-end balances. GL_BALANCE entity is mandatory for IFRS 9 reporting. |
| **BCBS 239 Principle 2 (Accuracy)** | Data reported to management/regulators must be accurate and reliable. Wrong source mappings (adjustment_id, posting_rule, central_bank_code) directly violate this principle. |
| **BCBS 239 Principle 6 (Adaptability)** | Risk data aggregation must work across organizational hierarchies. GL account hierarchy is required. |
| **UAE FTA — VAT Federal Decree-Law No. 8 of 2017** | Standard VAT rate 5%. Financial services are generally exempt. VAT_RULES entity must accurately reflect FTA-registered rates, not loan-agreement-level rates. |
| **CBUAE Regulatory Returns** | Central Bank codes (e.g., B201, T401) are defined in CBUAE's Statistical Reporting Framework. Must be sourced from the authoritative CBUAE code register, not from SWIFT/bank identifier tables. |
| **Finacle GL Architecture** | GL_SUB_HEAD_CODE (5-char) is the GL sub-head code; ACID (11-char) is the account identifier. These are distinct and non-interchangeable in Finacle's data model. |

---

*Report generated: 2026-04-27 | Assessor: Logical Model Assessor | Version: 1.0*
*Total findings: 41 (P0: 14, P1: 18, P2: 9)*
