# Silver-Layer Gap Assessment — Deposit Subject Area
**Repository:** imgokhan-rakbank/logical-model-assessment  
**Assessed SA:** `silver.deposit` (Priority 2 — High)  
**Assessment Date:** 2025-07-01  
**Assessor:** Silver-Layer Gap Assessment Agent  
**Guideline Version:** Data Platform Development Standards V1.0 Draft (December 2025)

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

| Dimension | Status | Score |
|---|---|---|
| Overall Health | 🔴 Critical | 28 / 100 |
| Identity Strategy (SK/BK) | 🔴 Missing | 0 / 10 |
| SCD-2 History | 🔴 Missing on most entities | 2 / 10 |
| Monetary Type Compliance | 🔴 All amounts as STRING | 0 / 10 |
| Schema Scoping | 🔴 Multiple wrong-SA entities | 3 / 10 |
| Mapping Completeness | 🟡 Partial | 6 / 10 |
| Metadata / Audit Columns | 🟡 Partial (some present) | 5 / 10 |
| Naming Conventions | 🟡 Mostly compliant | 6 / 10 |
| DQ Gate Compliance | 🔴 No quarantine tables visible | 2 / 10 |
| Reference Data Isolation | 🔴 Reference tables in wrong SA | 2 / 10 |

**Overall Assessment:** The Deposit subject area is **not production-ready** as a Silver-layer silver-standard implementation. The model lacks surrogate keys (`_sk`/`_bk`) on all entities, monetary amounts are stored as STRING throughout, SCD-2 columns are absent from the logical model for the majority of entities, and numerous tables belong to the wrong subject area. The largest physical table (`deposit_account`, 33.9 GB, 4.1 M files) is the most important asset but carries none of the required Silver metadata or identity columns per the logical model. Immediate remediation is required before any Gold-layer consumption can be trusted.

### 1.2 Top 5 Immediate Actions

| # | Action | Rule | Priority | Effort |
|---|---|---|---|---|
| 1 | Add `<entity>_sk` (MD5) and `<entity>_bk` to every logical entity and backfill physical tables | SLV-001, SLV-002 | P0 | High (6–8 weeks) |
| 2 | Add SCD-2 columns (`effective_from`, `effective_to`, `is_current`) to all entity tables; reprocess history from Bronze | SLV-003, §2.3 | P0 | High (6–8 weeks) |
| 3 | Migrate all monetary attributes from STRING to `DECIMAL(18,4)` (or BIGINT fils for AED) | SLV-010, §Gate 2 | P0 | Medium (3–4 weeks) |
| 4 | Relocate out-of-scope entities: move Collateral, GL, Reference Data, Payments entities to their respective SAs | SA scoping, §2.2 | P0 | Medium (2–3 weeks) |
| 5 | Remove or archive temp/legacy tables (`deposit_account_old`, `*_temp_no_partition`, `deposits_kvp`) and fix `deffered_income_tax_account` typo | NAM-001, §2.11 | P1 | Low (1 week) |

---

## 2. Subject Area Architecture Assessment

### 2.1 Correct Scoping per `sa/subject-areas.md`

**Status: 🔴 Non-Compliant**

The `sa/subject-areas.md` definition for Deposit (SA #6) states:

> `silver.deposit` — CASA and Term Deposit Details: savings, current, and fixed-term accounts including interest/profit schedules, rollover instructions, and early-breakage events.

The current physical schema `silver_dev.deposits` contains **26 tables**, many of which are out of scope:

| Table | Expected SA | Issue |
|---|---|---|
| `collateral_item` | `silver.collateral` | Belongs to Lending/Collateral SA |
| `collateral_item_class_association` | `silver.collateral` | Belongs to Lending/Collateral SA |
| `deffered_income_tax_account` | `silver.gl` | GL entity; also has a typo |
| `deposit_gl_account` | `silver.gl` | GL linkage belongs in Finance GL SA |
| `financial_event` | `silver.transaction` | Atomic financial posting — Transaction SA |
| `fund_transfer_event` | `silver.payment` | Fund transfer belongs in Payments SA |
| `interbank_event` | `silver.payment` | Interbank settlement — Payments SA |
| `bofd_check_event` | `silver.payment` | BOFD check clearing — Payments SA |
| `bofd_check_image` | `silver.payment` | BOFD document — Payments SA |
| `reference_currency` | `silver.reference` | ISO currency master — Reference Data SA |
| `reference_currency_code` | `silver.reference` | Duplicate currency reference — Reference Data SA |
| `reference_exchange_rate` | `silver.reference` | FX rates — Reference Data SA |
| `reference_transaction_code` | `silver.reference` | Transaction code lookup — Reference Data SA |
| `deposits_kvp` | None / Bronze | "Created by file upload UI" — not a managed Silver table |
| `deposit_balance_report` | Gold | Pre-aggregated report; violates SLV-006/SLV-007 |
| `deposit_account_old` | Deprecated | Legacy artifact; must be archived |
| `electronic_channel` | `silver.event` or `silver.reference` | Channel master belongs in Organisation or Reference SA |

**Entities correctly scoped:** `deposit_account`, `deposit_term_account`, `deposit_transaction_account`, `deposit_slip`, `safe_deposit_account`, `term_deposit_transaction`

**Confidence:** 0.95  
**Finding ID:** ARCH-001  
**Priority:** P0 | **Criticality:** High

---

### 2.2 Domain Decomposition vs BIAN / IFW

**Status: 🟡 Partially Aligned**

The core BIAN service domain for deposits is **"Deposit Account"** (BIAN v11 SD-017), which covers:
- Deposit Account Facility — opening, maintenance, closure
- Term Deposit — placement, rollover, early redemption
- CASA — current and savings accounts
- Interest/Profit Management (Islamic: Wakala, Mudaraba)
- Safe Deposit Box — vault services

**Alignment gaps:**
1. **No `deposit_interest_schedule` entity** — Interest/profit schedules mentioned in the SA description are absent from both logical model and physical schema.
2. **No `deposit_rollover_instruction` entity** — Rollover instructions (central to term deposits) are not modeled.
3. **No `deposit_early_redemption` entity** — Early-breakage events referenced in the SA description are absent.
4. **No Islamic profit rate entity** — For Islamic savings (Wakala/Mudaraba), the profit distribution mechanism is not captured; the `is_islamic_product` flag referenced in `sa/subject-areas.md` is absent from all deposit entities.
5. **CASA and Term Deposit merged** — IFW best practice separates CASA (Current Account Savings Account) from Term Deposit as distinct entity hierarchies. The current model conflates them in `deposit_account`.
6. **Transactions in Deposit SA** — `term_deposit_transaction` should be in `silver.transaction`; keeping it here creates dual-truth for transactional data.

**Confidence:** 0.90  
**Finding ID:** ARCH-002  
**Priority:** P1 | **Criticality:** High

---

### 2.3 Identity Strategy (SK/BK)

**Status: 🔴 Non-Compliant — Critical**

**Rule SLV-001:** Every Silver entity must have `<entity>_sk` (MD5 surrogate key).  
**Rule SLV-002:** Every Silver entity must have `<entity>_bk` (business/natural key).

**Findings:**
- Zero (0) of the 22 logical entities in the Deposit subject area define an `<entity>_sk` or `<entity>_bk` attribute in the logical model.
- The physical structures CSV does not expose column-level DDL; however, the data mapping workbook confirms that primary keys are expressed as raw source fields (e.g., `ACID`, `ACCOUNT_NUM_FORACID`, `TRAN_ID`) rather than canonical business keys with MD5 surrogate derivation.
- Without `_sk` columns, Gold-layer pipelines cannot build dimension tables or resolve SCD-2 history correctly.
- Without `_bk` columns, deduplication logic has no canonical key to operate on.

**Impact:** All downstream Gold pipelines are at risk of incorrect key derivation if they are not already broken. BCBS 239 lineage cannot be established.

**Confidence:** 0.98  
**Finding ID:** ARCH-003  
**Priority:** P0 | **Criticality:** High

---

### 2.4 SCD Strategy Coherence

**Status: 🔴 Non-Compliant — Critical**

**Rule SLV-003:** SCD-2 is the default change-tracking strategy. Tables may only use SCD-1 with explicit data-steward approval.

**Findings:**
- The logical model does not include `effective_from`, `effective_to`, or `is_current` attributes for the majority of entities.
- Only entities with `Business Start Date`, `Business End Date`, and `Is Active Flag` attributes (`Deposit Account`, `Deposit Term Account`, `Deposit Slip`, `Term Deposit Transaction`) have any lifecycle tracking — and these use `CHAR(18)` type for date fields, which is incorrect (should be `TIMESTAMP`).
- The `Business Start Date` / `Business End Date` pattern is a non-standard SCD-2 variant; the guideline mandates `effective_from` / `effective_to` / `is_current` (guideline §2.7).
- Entities with zero lifecycle columns: Account Event, BOFD Check Event, BOFD Check Image, Collateral Item, Collateral Item Class Association, Contact Event, Deffered Income Tax Account, Deposit GL Account, Electronic Channel, Financial Event, Fund Transfer Event, InterBank Event, Reference_ATM, Reference_Currency, Reference_Exchange_Rate, Reference_Transaction_Code, Safe Deposit Account.

**Confidence:** 0.97  
**Finding ID:** ARCH-004  
**Priority:** P0 | **Criticality:** High

---

### 2.5 Cross-Entity Referential Integrity

**Status: 🔴 Non-Compliant**

**Findings:**
1. **No FK attributes defined in the logical model** — No entity contains a `<related_entity>_sk` or `<related_entity>_bk` FK attribute linking to another entity within or outside the SA.
2. **`deposit_account` → `silver.party`**: The deposit account must reference a customer/party (`customer_bk`). This FK is absent from the logical model.
3. **`deposit_account` → `silver.account`**: Per SA taxonomy, Account SA holds "Unified Balance records for CASA, Term Deposits". The deposit SA should carry an `account_bk` FK. This is absent.
4. **`deposit_account` → `silver.product`**: No product FK links the deposit account to its product definition.
5. **`term_deposit_transaction` → `deposit_account`**: The transaction entity references account via raw `Account Number` string but no SK-based FK.
6. **`deposit_term_account` → `deposit_account`**: Logical parent-child relationship is implied but not declared as a FK.
7. **`safe_deposit_account` → `deposit_account`**: Similar parent-child relationship undeclared.
8. **`deposit_slip` → `deposit_account`**: No FK declared.

**Confidence:** 0.95  
**Finding ID:** ARCH-005  
**Priority:** P0 | **Criticality:** High

---

### 2.6 ETL Mapping Completeness and Lineage

**Status: 🟡 Partially Complete**

**Summary of mapping coverage:**

| Entity | Total Attributes (Logical) | Mapped | Not Available | Coverage % |
|---|---|---|---|---|
| Account Event | 10 | 4 | 0 | 40% |
| BOFD Check Event | 9 | 2 | 1 | 22% |
| BOFD Check Image | 10 | 3 | 1 | 30% |
| Collateral Item | 9 | 1 | 2 | 11% |
| Collateral Item Class Association | 11 | 3 | 2 | 27% |
| Contact Event | 15 | 4 | 5 | 27% |
| Deffered Income Tax Account | 7 | 1 | 0 | 14% |
| Deposit Account | 17 | 5 | 2 | 29% |
| Deposit GL Account | 7 | 1 | 0 | 14% |
| Deposit Slip | 18 | 6 | 2 | 33% |
| Deposit Term Account | 13 | 4 | 0 | 31% |
| Deposit Transaction Account | 8 | 2 | 0 | 25% |
| Electronic Channel | 2 | 1 | 1 | 50% |
| Financial Event | 12 | 9 | 3 | 75% |
| Fund Transfer Event | 7 | 7 | 0 | 100% |
| InterBank Event | 4 | 0 | 4 | 0% |
| Reference_ATM | 14 | 5 | 3 | 36% |
| Reference_Currency | 11 | 0 | 0 | 0% (not in mapping) |
| Reference_Exchange_Rate | 13 | 5 | 0 | 38% |
| Reference_Transaction_Code | 24 | 11 | 0 | 46% |
| Safe Deposit Account | 4 | 2 | 2 | 50% |
| Term Deposit Transaction | 37 | 28 | 5 | 76% |

**Additional entities in mapping not in logical model:**
- `Financial_Agreement` — 9 attributes, 8 mapped (no corresponding logical entity)
- `Bank Event` — 1 attribute (no corresponding logical entity)
- `Reference_Channel` — 7 attributes (no corresponding logical entity)
- `Reference_Currency_Code` — 4 attributes (separate from `Reference_Currency`)
- `ATM_Channel` / `REFERENCE_ATM` — overlap

**Critical mapping concerns:**
- `InterBank Event` has 0 mapped attributes (all "Not Available") — physical table `interbank_event` has 0 files/bytes, confirming no data flows.
- `fund_transfer_event` physical table also has 0 files/bytes despite 7/7 mapped attributes.
- Mapping source for `Account Modifier Number` (Account Event entity) is `CUSTOM.CUSTOM_STATIC_UPDATE.MODIFIED_USERID` — semantically this is a user ID, not an account modifier number; the mapping appears incorrect.
- `Deposit_Account.Document ID` has `Status: None` (neither Mapped nor Not Available) — unresolved.
- `Financial_Agreement` appears in the mapping but not in the logical model — orphaned mapping.

**Confidence:** 0.88  
**Finding ID:** ARCH-006  
**Priority:** P1 | **Criticality:** High

---

## 3. Entity Inventory

| # | Logical Entity | Physical Table | Status | Physical Files | Physical Size | SK Present | BK Present | SCD-2 Cols | Mapping % | Priority |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | Account Event | — | Unmapped (no physical table) | — | — | ❌ | ❌ | ❌ | 40% | P1 |
| 2 | BOFD Check Event | `bofd_check_event` | Mapped | 24 | 58 KB | ❌ | ❌ | ❌ | 22% | P1 |
| 3 | BOFD Check Image | `bofd_check_image` | Mapped | 21 | 164 MB | ❌ | ❌ | ❌ | 30% | P1 |
| 4 | Collateral Item | `collateral_item` | Out-of-Scope SA | 1 | 21 KB | ❌ | ❌ | ❌ | 11% | P0 (relocate) |
| 5 | Collateral Item Class Association | `collateral_item_class_association` | Out-of-Scope SA | 13 | 8.5 MB | ❌ | ❌ | ❌ | 27% | P0 (relocate) |
| 6 | Contact Event | — | Unmapped | — | — | ❌ | ❌ | ❌ | 27% | P1 |
| 7 | Deffered Income Tax Account | `deffered_income_tax_account` | Out-of-Scope SA + Typo | 1 | 6 MB | ❌ | ❌ | ❌ | 14% | P0 (relocate) |
| 8 | Deposit Account | `deposit_account` | Core — Incomplete | 4,112,256 | 33.9 GB | ❌ | ❌ | Partial | 29% | P0 |
| 9 | Deposit GL Account | `deposit_gl_account` | Out-of-Scope SA | 3 | 30 KB | ❌ | ❌ | ❌ | 14% | P0 (relocate) |
| 10 | Deposit Slip | `deposit_slip` | Empty Table | 0 | 0 | ❌ | ❌ | Partial | 33% | P1 |
| 11 | Deposit Term Account | `deposit_term_account` | Core — Incomplete | 3 | 870 KB | ❌ | ❌ | Partial | 31% | P0 |
| 12 | Deposit Transaction Account | `deposit_transaction_account` | Core — Incomplete | 1 | 160 KB | ❌ | ❌ | ❌ | 25% | P1 |
| 13 | Electronic Channel | `electronic_channel` | Out-of-Scope SA | 1 | 2.4 KB | ❌ | ❌ | ❌ | 50% | P1 (relocate) |
| 14 | Financial Event | `financial_event` | Out-of-Scope SA | 1 | 5 KB | ❌ | ❌ | ❌ | 75% | P0 (relocate) |
| 15 | Fund Transfer Event | `fund_transfer_event` | Out-of-Scope SA + Empty | 0 | 0 | ❌ | ❌ | ❌ | 100% | P1 (relocate) |
| 16 | InterBank Event | `interbank_event` | Out-of-Scope SA + Empty | 0 | 0 | ❌ | ❌ | ❌ | 0% | P1 (relocate) |
| 17 | Reference_ATM | — | Out-of-Scope SA | — | — | ❌ | ❌ | ❌ | 36% | P1 (relocate) |
| 18 | Reference_Currency | `reference_currency` | Out-of-Scope SA | 1 | 7.7 KB | ❌ | ❌ | ❌ | 0% | P0 (relocate) |
| 19 | Reference_Exchange_Rate | `reference_exchange_rate` | Out-of-Scope SA | 2 | 9.9 KB | ❌ | ❌ | ❌ | 38% | P0 (relocate) |
| 20 | Reference_Transaction_Code | `reference_transaction_code` | Out-of-Scope SA | 584 | 3 MB | ❌ | ❌ | ❌ | 46% | P0 (relocate) |
| 21 | Safe Deposit Account | `safe_deposit_account` | Core — Incomplete | 5 | 31 MB | ❌ | ❌ | ❌ | 50% | P1 |
| 22 | Term Deposit Transaction | `term_deposit_transaction` | Boundary Dispute | 916,617 | 7.6 GB | ❌ | ❌ | Partial | 76% | P0 |
| — | `deposit_account_old` | Legacy artifact | — | 2 | 38 MB | — | — | — | — | P1 (archive) |
| — | `deposit_balance_report` | Aggregated report violation | — | 1 | 4.2 KB | — | — | — | — | P0 (remove) |
| — | `deposits_kvp` | Unmanaged upload artifact | — | 1 | 12.6 KB | — | — | — | — | P1 (remove) |
| — | `reference_currency_code` | Duplicate reference (out-of-scope) | — | 1 | 6 KB | — | — | — | — | P1 (consolidate/relocate) |
| — | `*_temp_no_partition` (×3) | Temp tables in production | — | 0–1 | various | — | — | — | — | P1 (remove) |

---

## 4. Per-Entity Assessments

### 4.1 Entity: Account Event

**Logical Definition:** Not explicitly defined (minimal definition in model); represents events occurring on an account (charges, modifications).  
**Physical Table:** None found in `silver_dev.deposits`  
**Confidence:** 0.80

#### 4.1a — Industry Fitness

The "Account Event" concept is analogous to the BIAN **"Account Event Notification"** service operation. In IFW, this maps to the Account Event subtype of the Party Event entity. However, its placement in the Deposit SA is questionable — account events span all product types (loans, cards, deposits), making it a cross-SA concern that should reside in either the Account SA or the Business Event SA.

**Issues:**
- **Boundary violation**: Account events are not deposit-specific; they should be in `silver.account` or `silver.event`.
- **No physical table**: The entity exists in the logical model but has no physical implementation.
- **Scope underspecification**: The entity has only 10 attributes, of which only 4 are mapped; the definition of "Account Event Type Code" is left unmapped.

#### 4.1b — Attribute-Level Review

| Attribute | Type (Logical) | Mapping Status | Source | Issues |
|---|---|---|---|---|
| Event Id | STRING | Mapped | `TBAADM.GEN_CHRG_HEADER_TBL.EVENT_ID` | Source is a charge header table — semantically ambiguous for generic "account event" |
| Account Number | STRING | Mapped | `CRMUSER.INCIDENTS.ACCOUNTNUMBER` | Source is CRM Incidents — not the canonical account master; wrong source table |
| Account Modifier Number | STRING | Mapped | `CUSTOM.CUSTOM_STATIC_UPDATE.MODIFIED_USERID` | **Critical semantic error**: MODIFIED_USERID is a user ID, not an account modifier number |
| Account Event Type Code | STRING | Mapped | `TBAADM.DAILY_TRAN_DETAIL_TABLE.PTTM_EVENT_TYPE` | Raw source code — no canonical mapping per SLV-004 |
| Source System Code | STRING | None | — | Required audit column |
| Source System Id | STRING | None | — | Required audit column |
| IsLogicallyDeleted | STRING | None | — | Non-standard column name; should be `is_active_flag` (NAM-001) |
| Create Date | STRING | None | — | Type should be TIMESTAMP, not STRING (SLV-009) |
| Update Date | STRING | None | — | Type should be TIMESTAMP, not STRING (SLV-009) |
| Deleted Date | STRING | None | — | Type should be TIMESTAMP, not STRING (SLV-009) |
| **entity_sk** | — | **MISSING** | — | **SLV-001: MD5 surrogate key absent** |
| **entity_bk** | — | **MISSING** | — | **SLV-002: Business key absent** |
| **effective_from** | — | **MISSING** | — | **SLV-003: SCD-2 column absent** |
| **effective_to** | — | **MISSING** | — | **SLV-003: SCD-2 column absent** |
| **is_current** | — | **MISSING** | — | **SLV-003: SCD-2 column absent** |

#### 4.1c — Metadata Completeness

All 11 mandatory Silver audit columns are absent from the logical model. The attributes `Create Date`, `Update Date`, `Deleted Date` approximate some audit columns but use incorrect types (STRING instead of TIMESTAMP) and incorrect names (`IsLogicallyDeleted` ≠ `is_active_flag`). The required columns `run_id`, `source_ingestion_date`, `effective_from`, `effective_to`, `is_current` are all absent.

**Findings:**
- F-001: No physical table (Criticality: High, Priority: P1)
- F-002: Wrong subject area (Criticality: Medium, Priority: P1)
- F-003: Semantic mapping error on `Account Modifier Number` (Criticality: High, Priority: P0)
- F-004: All SK/BK/SCD-2 columns missing (Criticality: High, Priority: P0)
- F-005: All date/timestamp fields typed as STRING (Criticality: High, Priority: P0)

---

### 4.2 Entity: BOFD Check Event

**Logical Definition:** Bank of First Deposit check clearing event; records check deposit events through electronic clearing channels.  
**Physical Table:** `silver_dev.deposits.bofd_check_event` (24 files, 58 KB)  
**Confidence:** 0.82

#### 4.2a — Industry Fitness

BOFD (Bank of First Deposit) is a check clearing concept from the US UCC/ECCHO framework, but also applicable in UAE CBUAE check truncation/clearing operations. In BIAN, this maps to the **"Payment Order"** service domain, not the Deposit domain. The entity describes a payment/clearing event and should be in `silver.payment`.

**Issues:**
- **Wrong subject area**: BOFD check events are payment clearing events, not deposit-product events.
- **Incomplete attributes**: Only 3 of 9 logical attributes appear in the mapping; 6 standard audit attributes (`Source System Code`, `Source System Id`, `IsLogicallyDeleted`, dates) are absent from the mapping.

#### 4.2b — Attribute-Level Review

| Attribute | Type (Logical) | Mapping Status | Source | Issues |
|---|---|---|---|---|
| Event Id | STRING | Mapped | `TBAADM.PAYMENT_ORDER_DETAIL_TABLE.CHRG_EVENT_ID` | Correct source for BOFD check; complex join required |
| Channel Id | STRING | Not Available | `CRMUSER.CHANNELS.CHANNELID` | Source identified but data not available |
| Channel Type Code | STRING | Mapped | `TBAADM.GENERAL_ACCT_MAST_TABLE.CHANNEL_LEVEL_CODE` | Raw source code — violates SLV-004 |
| Source System Code | STRING | None | — | Required audit column unmapped |
| Source System Id | STRING | None | — | Required audit column unmapped |
| IsLogicallyDeleted | STRING | None | — | Non-standard name; should be `is_active_flag` |
| Create Date | STRING | None | — | Type error: should be TIMESTAMP |
| Update Date | STRING | None | — | Type error: should be TIMESTAMP |
| Deleted Date | STRING | None | — | Type error: should be TIMESTAMP |
| **bofd_check_event_sk** | — | **MISSING** | — | SLV-001 |
| **bofd_check_event_bk** | — | **MISSING** | — | SLV-002 |
| **effective_from/to/is_current** | — | **MISSING** | — | SLV-003 |

#### 4.2c — Metadata Completeness

All mandatory Silver metadata columns are absent from the logical model. Physical table has 58 KB of data (likely reference/lookup rows), indicating the entity is populated but without proper metadata controls.

**Findings:**
- F-006: Wrong SA — belongs in `silver.payment` (Criticality: Medium, Priority: P1)
- F-007: All SK/BK/SCD-2 columns missing (Criticality: High, Priority: P0)
- F-008: Channel Type Code stores raw source code without canonical mapping (SLV-004) (Criticality: Medium, Priority: P1)

---

### 4.3 Entity: BOFD Check Image

**Logical Definition:** Digital image/document record of a Bank of First Deposit check.  
**Physical Table:** `silver_dev.deposits.bofd_check_image` (21 files, 164 MB)  
**Confidence:** 0.80

#### 4.3a — Industry Fitness

Check image storage is a document management / payment clearing function. In BIAN, this maps to **"Document Services"** or **"Payment Order"**. Like the BOFD Check Event, this entity does not belong in the Deposit SA.

The 164 MB physical size suggests actual check image metadata is being stored; given PII concerns (check images contain account numbers, signatures), this entity requires additional data classification controls per §10 (PII & Sensitive Data Controls).

#### 4.3b — Attribute-Level Review

| Attribute | Type (Logical) | Mapping Status | Source | Issues |
|---|---|---|---|---|
| Draft Document Id | STRING | Not Available | None | Primary key unavailable — critical gap |
| Payor Check Account Number | STRING | Mapped | `TBAADM.GENERAL_ACCT_MAST_TABLE.ACCT_NUM` | PII field — requires SHA-256 masking per Gate 2 controls |
| Related External UDK Number | STRING | Mapped | `TBAADM.PAYMENT_ORDER_DETAIL_TABLE.TRAN_REF_NUM` | Join-based transform |
| Payor Bank Check Txn Code Val | STRING | Mapped | `TBAADM.PAYMENT_ORDER_DETAIL_TABLE.TRAN_TYPE` | Raw source code — violates SLV-004 |
| Source System Code | STRING | None | — | Required audit column |
| Source System Id | STRING | None | — | Required audit column |
| IsLogicallyDeleted | STRING | None | — | Non-standard name |
| Create Date | STRING | None | — | Type: should be TIMESTAMP |
| Update Date | STRING | None | — | Type: should be TIMESTAMP |
| Deleted Date | STRING | None | — | Type: should be TIMESTAMP |

#### 4.3c — Metadata Completeness

All SK/BK/SCD-2 and standard metadata columns are absent. `Draft Document Id` (natural PK) is "Not Available" — meaning the entity has no reliable primary key from source.

**Findings:**
- F-009: Wrong SA — belongs in `silver.payment` (Medium, P1)
- F-010: Primary key (`Draft Document Id`) has no source mapping (High, P0)
- F-011: PII field `Payor Check Account Number` must be masked (High, P0)
- F-012: All SK/BK/SCD-2 missing (High, P0)

---

### 4.4 Entity: Collateral Item

**Logical Definition:** A collateral item pledged as security for a financial obligation.  
**Physical Table:** `silver_dev.deposits.collateral_item` (1 file, 21 KB)  
**Confidence:** 0.95

#### 4.4a — Industry Fitness

**This entity definitively belongs to `silver.collateral` (Lending SA, sub-area 8b)**. Collateral is security pledged against lending facilities, not deposit products. Its presence in the Deposit SA is a domain model error. BIAN service domain: **"Collateral Asset Administration"** (SD-024).

#### 4.4b — Attribute-Level Review

| Attribute | Type | Mapping Status | Source | Issues |
|---|---|---|---|---|
| Collateral Item ID | STRING | Mapped | `TBAADM.CHARGE_OFF_TABLE.ACID` | Source is a charge-off table; mapping is semantically questionable |
| Collateral Item Type Code | STRING | Not Available | None | No source identified |
| Collateral Item Host Id | STRING | Not Available | None | No source identified |
| Source System Code–Deleted Date | STRING | None | — | Standard audit columns unmapped |

#### 4.4c — Metadata Completeness

All metadata absent. 7/9 attributes are unmapped or have no source.

**Findings:**
- F-013: **Wrong SA — must be moved to `silver.collateral`** (High, P0)
- F-014: All SK/BK/SCD-2 missing (High, P0)
- F-015: Primary source mapping uses charge-off table — wrong data domain (High, P0)

---

### 4.5 Entity: Collateral Item Class Association

**Logical Definition:** Associates a collateral item with an asset class and value.  
**Physical Table:** `silver_dev.deposits.collateral_item_class_association` (13 files, 8.5 MB)  
**Confidence:** 0.95

#### 4.5a — Industry Fitness

Same scoping issue as Collateral Item — this is a `silver.collateral` entity. The 8.5 MB size indicates live data, meaning collateral data is currently incorrectly housed in the Deposits schema.

#### 4.5b — Attribute-Level Review

| Attribute | Type | Mapping Status | Source | Issues |
|---|---|---|---|---|
| Collateral Item ID | STRING | Mapped | `TBAADM.CHARGE_OFF_TABLE.ACID` | Questionable source (charge-off, not collateral master) |
| Party Asset Class Code | STRING | Not Available | None | No source |
| Collateral Item Class Start Date | STRING | Mapped | `TBAADM.CHARGE_OFF_TABLE.CHRGE_OFF_DATE` | Charge-off date ≠ class start date — semantic mismatch |
| Collateral Item Class End Date | STRING | Not Available | None | No source |
| Party Asset Value Code | STRING | Mapped | `TBAADM.CHARGE_OFF_TABLE.LCHG_USER_ID` | **Critical error**: LCHG_USER_ID is a user ID, not an asset value code |

#### 4.5c — Metadata Completeness

All metadata absent.

**Findings:**
- F-016: Wrong SA (High, P0)
- F-017: `Party Asset Value Code` mapped to `LCHG_USER_ID` — critical semantic error (High, P0)
- F-018: `Collateral Item Class Start Date` mapped to charge-off date — incorrect (High, P0)

---

### 4.6 Entity: Contact Event

**Logical Definition:** A customer contact interaction event through a channel.  
**Physical Table:** None found  
**Confidence:** 0.80

#### 4.6a — Industry Fitness

"Contact Event" is a Business Event / CRM concept, mapping to BIAN **"Customer Contact"** (SD-088). Per `sa/subject-areas.md`, contact center interactions belong in `silver.event` (Business Event SA #11). This entity is misclassified in the Deposit SA.

#### 4.6b — Attribute-Level Review

| Attribute | Type | Mapping Status | Issues |
|---|---|---|---|
| Event Id | STRING | mapped (lowercase) | Source from charge header table — semantically questionable |
| Channel Id | STRING | Mapped | ✓ |
| Channel Type Code | STRING | Not available | No source |
| Access Medium Type Code | STRING | Not available | No source |
| Language Code | STRING | Mapped | From `GENERAL_ACCT_MAST_TABLE.LANG_CODE` — account language, not contact language |
| Authorization Number | STRING | Mapped | From `CRMUSER.DEMOGRAPHIC.CONTACTID` — CONTACTID ≠ AuthorizationNumber (semantic error) |
| Promo Type Code | STRING | Not Available | No source |
| Direct Contact Ind | STRING | Not available | No source |
| Contact Initiation Type Code | STRING | Not available | No source |

#### 4.6c — Metadata Completeness

No physical table, no SK/BK/SCD-2, all standard metadata absent.

**Findings:**
- F-019: Wrong SA — belongs in `silver.event` (Medium, P1)
- F-020: No physical implementation (Medium, P1)
- F-021: `Authorization Number` mapped to `CONTACTID` — semantic error (High, P0)

---

### 4.7 Entity: Deffered Income Tax Account

**Logical Definition:** GL account used to track deferred income tax.  
**Physical Table:** `silver_dev.deposits.deffered_income_tax_account` (1 file, 6 MB)  
**Confidence:** 0.98

#### 4.7a — Industry Fitness

Deferred income tax is a **Finance GL / Accounting** concept. It belongs in `silver.gl` (Finance GL SA #19). It has no relationship to the deposit product lifecycle. Additionally, the entity name contains a spelling error: **"Deffered"** should be **"Deferred"** (NAM-001 violation).

#### 4.7b — Attribute-Level Review

| Attribute | Type | Mapping Status | Source | Issues |
|---|---|---|---|---|
| GL Main Account Num | STRING | Mapped | `TBAADM.GENERAL_ACCT_MAST_TABLE.ACCT_NUM` | Mapped but no GL context; account master is not the GL sub-head table |

Only 1 of 7 attributes is in the mapping document. The 6 standard audit attributes are unmapped.

#### 4.7c — Metadata Completeness

All metadata absent.

**Findings:**
- F-022: Wrong SA — must move to `silver.gl` (High, P0)
- F-023: Entity name typo `deffered` → `deferred` (NAM-001) (Low, P1)
- F-024: Only 1/7 attributes mapped (High, P0)
- F-025: All SK/BK/SCD-2 missing (High, P0)

---

### 4.8 Entity: Deposit Account

**Logical Definition:** A customer deposit account combining Conventional Banking (Finacle) and Islamic Banking (Flexcube) data, including maturity code, interest disbursement type, and original deposit amount.  
**Physical Table:** `silver_dev.deposits.deposit_account` (4,112,256 files, 33.9 GB — the largest and most critical table)  
**Confidence:** 0.95

#### 4.8a — Industry Fitness

`deposit_account` is correctly scoped within the Deposit SA. It maps to BIAN **"Deposit Account Facility"** (SD-017, service operation "Initiate"). However, the entity is critically under-modeled:

1. **CASA vs Term Deposit conflation**: The entity conflates savings/current (CASA) and term deposit accounts into a single table. IFW best practice separates these since they have materially different attributes (e.g., CASA has no maturity date; TD has maturity, rollover, and interest disbursement schedules).
2. **Missing balance attributes**: The entity has no balance-related attributes (opening balance, closing balance, available balance). These are core to a deposit account entity per BIAN.
3. **Missing account status**: No `account_status_code` attribute — critical for lifecycle management.
4. **Missing party linkage**: No `customer_bk` or `party_sk` FK attribute.
5. **Missing product linkage**: No `product_bk` or `product_sk`.
6. **Islamic product attributes absent**: No `is_islamic_product` flag, `profit_rate`, or `sharia_structure_code`.
7. **4.1 million files**: This is a severe file fragmentation issue (average ~8 KB per file). Delta Lake `OPTIMIZE` and `ZORDER` operations are critically needed. This is a platform engineering concern.

#### 4.8b — Attribute-Level Review

| Attribute | Type (Logical) | Mapping Status | Source | Issues |
|---|---|---|---|---|
| Account Number | STRING | Mapped | `CUSTOM.TD_ACCOUNTS.ACCOUNT_NUM_FORACID` | Business key candidate — should become `deposit_account_bk` |
| Account Modifier Number | STRING | Mapped | `CUSTOM.TD_ACCOUNTS.ACID` | Internal system key; may be the true BK |
| Maturity Cd | STRING | Not available | None | Critical for TD — no source mapping |
| Interest DisbmType Code | STRING | Not available | None | Critical for TD — no source; abbreviation `DisbmType` violates NAM-001 |
| Original Deposit Amt | STRING | Mapped | `CUSTOM.TD_ACCOUNTS.ORIGINAL_DEPOSIT_AMOUNT` | **SLV-010**: Amount as STRING; should be DECIMAL(18,4) or BIGINT (fils) |
| Acct Currency Original Deposit Amt | STRING | Mapped | `CUSTOM.TD_ACCOUNTS.ACCT_CRNCY_CODE` | Attribute name is overly long; mixed semantic (currency code, not amount) |
| Deposit creation channel id | (None) | Mapped | `TBAADM.GENERAL_ACCT_MAST_TABLE.CHANNEL_ID` | Data type not defined in mapping; source has typo "Finalce" |
| Document Id | (None) | None (Status: None) | — | Undefined status; not resolved |
| Business Start Date | CHAR(18) | — | — | **SCD-2 proxy** — wrong type (CHAR not TIMESTAMP); non-standard column name |
| Business End Date | CHAR(18) | — | — | Same issue |
| Is Active Flag | CHAR(18) | — | — | Type CHAR(18) for a boolean flag is incorrect; should be BOOLEAN or STRING 'Y'/'N' |
| **deposit_account_sk** | — | **MISSING** | — | SLV-001 |
| **deposit_account_bk** | — | **MISSING** | — | SLV-002 |
| **effective_from** | — | **MISSING** | — | SLV-003 |
| **effective_to** | — | **MISSING** | — | SLV-003 |
| **is_current** | — | **MISSING** | — | SLV-003 |
| **customer_bk / party_sk** | — | **MISSING** | — | No party FK |
| **account_status_code** | — | **MISSING** | — | Critical business attribute absent |
| **product_bk** | — | **MISSING** | — | No product FK |

#### 4.8c — Metadata Completeness

`Business Start Date`, `Business End Date`, `Is Active Flag` partially proxy for lifecycle metadata but are typed as CHAR(18) — incorrect. The 11 mandatory Silver audit columns (`source_system_code`, `source_system_id`, `create_date`, `update_date`, `delete_date`, `is_active_flag`, `effective_from`, `effective_to`, `is_current`, `run_id`, `source_ingestion_date`) are not defined in the logical model.

**Findings:**
- F-026: No `deposit_account_sk` / `deposit_account_bk` (Critical, P0)
- F-027: No SCD-2 columns (`effective_from/to/is_current`) — `Business Start/End Date` typed as CHAR(18) (Critical, P0)
- F-028: `Original Deposit Amt` stored as STRING; must be DECIMAL(18,4) (High, P0)
- F-029: Missing `account_status_code`, `customer_bk`, `product_bk` (High, P0)
- F-030: 4.1M files — critical file fragmentation; requires OPTIMIZE (Medium, P1)
- F-031: CASA and Term Deposit should be separate entities (Medium, P1)
- F-032: `Interest DisbmType Code` name violates NAM-001 (abbreviation) (Low, P2)
- F-033: No Islamic product attributes despite dual-system sourcing (Medium, P1)

---

### 4.9 Entity: Deposit GL Account

**Logical Definition:** General ledger account associated with deposit accounts.  
**Physical Table:** `silver_dev.deposits.deposit_gl_account` (3 files, 30 KB)  
**Confidence:** 0.95

#### 4.9a — Industry Fitness

A GL account entity belongs in `silver.gl` (Finance GL SA #19). Per the SA taxonomy, the GL SA covers "Chart of Accounts, General Ledger, Trial Balances". Having a GL account entity in the Deposit SA violates `silver.deposit` scoping and creates a risk of conflicting GL definitions between the two SAs.

#### 4.9b — Attribute-Level Review

| Attribute | Type | Mapping Status | Source | Issues |
|---|---|---|---|---|
| GL Main Account Num | STRING | Mapped | `TBAADM.EOD_TRAN_DETAIL_TABLE.GL_SUB_HEAD_CODE` | GL sub-head code from EOD transactions — not the GL account master |

Only 1/7 attributes mapped. The entity is a stub with no useful semantic content beyond a single GL code.

#### 4.9c — Metadata Completeness

All metadata absent.

**Findings:**
- F-034: Wrong SA — must move to `silver.gl` (High, P0)
- F-035: 6/7 attributes unmapped (Medium, P1)

---

### 4.10 Entity: Deposit Slip

**Logical Definition:** A physical/electronic deposit slip document recording a deposit transaction's details.  
**Physical Table:** `silver_dev.deposits.deposit_slip` (0 files, 0 bytes — empty)  
**Confidence:** 0.85

#### 4.10a — Industry Fitness

A Deposit Slip is a valid deposit SA entity — it records the physical evidence of a deposit being made. It maps to BIAN **"Deposit Account Facility"** service operation "Book". However, it overlaps with BOFD Check Image for check deposits; the scoping boundary between these two entities is unclear.

#### 4.10b — Attribute-Level Review

| Attribute | Type | Mapping Status | Source | Issues |
|---|---|---|---|---|
| Document Id | STRING | Not Available | None | PK has no source — critical; physical table is empty likely for this reason |
| External UDK num | STRING | Mapped | `TBAADM.PAYMENT_ORDER_DETAIL_TABLE.TRAN_REF_NUM` | Correct source |
| Deposit Slip Txn Type Code | STRING | Mapped | `TBAADM.PAYMENT_ORDER_DETAIL_TABLE.TRAN_TYPE` | Raw source code — SLV-004 |
| Issuing Bank Id | STRING | Mapped | `TBAADM.PAYMENT_ORDER_DETAIL_TABLE.BANK_ID` | ✓ |
| Deposit Check Serial Num | STRING | Mapped | `TBAADM.PAYMENT_ORDER_DETAIL_TABLE.SRL_NUM` | ✓ |
| Deposit Slip Entered Date | STRING | Mapped | `.CR_EXEC_DATE` | Type: should be DATE, not STRING |
| Deposit Slip Amt | STRING | Mapped | `.REMIT_AMT` | **SLV-010**: Amount as STRING; must be DECIMAL(18,4) |
| Document Image Source Code | STRING | Not Available | None | No source |
| Document Image Type Code | STRING | Not Available | None | No source |
| Business Start/End Date | CHAR(18) | — | — | SCD-2 proxy — wrong type |
| Is Active Flag | CHAR(18) | — | — | Wrong type |

#### 4.10c — Metadata Completeness

Table is empty (0 bytes) — likely never populated. Standard metadata absent.

**Findings:**
- F-036: Physical table empty — PK (`Document Id`) has no source mapping (High, P0)
- F-037: `Deposit Slip Amt` as STRING (High, P0)
- F-038: `Deposit Slip Entered Date` as STRING instead of DATE (Medium, P1)
- F-039: All SK/BK/SCD-2 missing (High, P0)

---

### 4.11 Entity: Deposit Term Account

**Logical Definition:** Extension of a deposit account for term (fixed) deposits; captures maturity date and grace period.  
**Physical Table:** `silver_dev.deposits.deposit_term_account` (3 files, 870 KB)  
**Confidence:** 0.90

#### 4.11a — Industry Fitness

This entity correctly captures term-deposit-specific attributes. However, it is severely under-modeled — a term deposit requires many more attributes:
- Interest rate / profit rate
- Rollover type (auto/manual)
- Rollover instructions
- Early redemption penalty rate
- Interest disbursement method
- Compounding frequency

The current 4-mapped-attribute model provides only maturity date and grace period.

#### 4.11b — Attribute-Level Review

| Attribute | Type | Mapping Status | Source | Issues |
|---|---|---|---|---|
| Account Number | STRING | Mapped | `TBAADM.TD_ACCT_MASTER_TABLE.ACID` | ACID is an internal system key; should be `deposit_account_bk` FK |
| Account Modifier Number | STRING | Mapped | `TBAADM.TD_ACCT_MASTER_TABLE.ACID` | Same source as Account Number — duplicate mapping |
| Next Term Maturity Date | STRING | Mapped | `TBAADM.TD_ACCT_MASTER_TABLE.MATURITY_DATE` | Type: should be DATE not STRING |
| Grace Period End Date | STRING | Mapped | `TD_ACCT_MASTER_TABLE.NUM_OF_GRACE_DAYS_UTIL` | **Critical semantic error**: `NUM_OF_GRACE_DAYS_UTIL` is a number of days, not a date; mapping a count as a date column is incorrect |
| Business Start/End Date | CHAR(18) | — | — | SCD-2 proxy — wrong type |
| Is Active Flag | CHAR(18) | — | — | Wrong type |

#### 4.11c — Metadata Completeness

All mandatory metadata absent.

**Findings:**
- F-040: `Grace Period End Date` mapped to `NUM_OF_GRACE_DAYS_UTIL` — number ≠ date (Critical semantic error) (High, P0)
- F-041: `Account Number` and `Account Modifier Number` both map to same source column `ACID` — duplicate (Medium, P1)
- F-042: Missing interest/profit rate, rollover attributes (Medium, P1)
- F-043: All SK/BK/SCD-2 missing (High, P0)
- F-044: Date fields stored as STRING (Medium, P1)

---

### 4.12 Entity: Deposit Transaction Account

**Logical Definition:** Associates a deposit account with its transaction activity.  
**Physical Table:** `silver_dev.deposits.deposit_transaction_account` (1 file, 160 KB)  
**Confidence:** 0.75

#### 4.12a — Industry Fitness

This entity appears to be a bridge/associative entity between deposit accounts and transactions. Its existence is questionable — in a properly normalized 3NF model, the relationship between an account and its transactions is captured via an FK on the transaction entity. A separate bridge table is unnecessary unless there is a many-to-many relationship, which is not the case here.

In BIAN, deposit transactions are handled by the **"Deposit Account Fulfillment Arrangement"** service operation, not a separate bridge entity.

#### 4.12b — Attribute-Level Review

| Attribute | Type | Mapping Status | Source | Issues |
|---|---|---|---|---|
| Account Number | STRING | Mapped | `CUSTOM.TD_ACCOUNTS.ACCOUNT_NUM_FORACID` | ✓ |
| Account Modifier Number | STRING | Mapped | `TBAADM.TD_TRAN_TABLE.ACID` | Different source table from Account Number — join required |

Only 2 attributes defined. No PK, no meaningful business attributes beyond account number linkage. This is essentially a redundant entity.

#### 4.12c — Metadata Completeness

All metadata absent.

**Findings:**
- F-045: Entity appears redundant — FK on `term_deposit_transaction.account_number` would suffice (Medium, P2)
- F-046: Only 2 attributes — insufficient entity definition (Medium, P1)
- F-047: All SK/BK/SCD-2 missing (High, P0)

---

### 4.13 Entity: Electronic Channel

**Logical Definition:** A digital/electronic channel through which deposit transactions are executed.  
**Physical Table:** `silver_dev.deposits.electronic_channel` (1 file, 2.4 KB)  
**Confidence:** 0.85

#### 4.13a — Industry Fitness

A channel reference entity belongs in either `silver.reference` (Reference Data SA) or `silver.event` / `silver.org` (Organisation SA). Per `sa/subject-areas.md`, `silver.org` covers "Branches, Departments, Cost Centers, Regions, ATMs, and Physical Assets". The channel master should reside there, not in Deposit SA.

#### 4.13b — Attribute-Level Review

| Attribute | Type | Mapping Status | Source | Issues |
|---|---|---|---|---|
| Channel Id | STRING | Mapped | `TBAADM.GENERAL_ACCT_MAST_TABLE.CHANNEL_ID` | Derived from account master — not a channel master table |
| Channel Type Code | STRING | Not available | None | No source for type code |

Only 2 attributes — severely under-modeled for a channel reference entity. No channel name, description, status, or active dates.

#### 4.13c — Metadata Completeness

The physical table has 2.4 KB of data. No metadata columns defined.

**Findings:**
- F-048: Wrong SA — belongs in `silver.org` or `silver.reference` (Medium, P1)
- F-049: Only 2 attributes; missing channel name, status (Low, P2)

---

### 4.14 Entity: Financial Event

**Logical Definition:** A financial event recording currency amounts, local/global amounts, period dates, and COA (Chart of Accounts) linkage.  
**Physical Table:** `silver_dev.deposits.financial_event` (1 file, 5 KB)  
**Confidence:** 0.85

#### 4.14a — Industry Fitness

"Financial Event" is a generic transactional event concept. In BIAN, atomic financial postings belong to **"General Ledger"** (GL) or **"Financial Accounting"** service domains — or in the Transaction SA (`silver.transaction`). Having financial events in the Deposit SA means that:
1. Transaction SA cannot provide a complete view of financial events without joining to the Deposit SA (violates §2.2 subject-area isolation).
2. GL-related attributes (`Financial Event COA`, `Ledger Batch Id`) indicate GL SA ownership.

The fact that the physical table has only 5 KB of data (near-empty) further suggests this entity is not being properly populated.

#### 4.14b — Attribute-Level Review

| Attribute | Type | Mapping Status | Source | Issues |
|---|---|---|---|---|
| Event Id | STRING | mapped | `TBAADM.MULTI_CRNCY_TRAN_DET_TBL.EVENT_ID` | ✓ |
| Event Currency Code | STRING | Mapped | `.CHRG_CRNCY_FROM` | Source column name "from" suggests this is the source currency, not event currency |
| Event Amt | STRING | Mapped | `.EVENT_AMT` | **SLV-010**: Amount as STRING |
| Event Local Amt | STRING | Mapped | `TBAADM.COLLECTION_HIST_DET_TBL.NOSTRO_AMT` | Nostro amount from collection history ≠ local deposit amount — wrong source |
| Event Global Amt | STRING | Mapped | `TBAADM.COLLECTION_HIST_DET_TBL.CONT_LIAB_AMT` | Contingent liability from collections ≠ global event amount |
| Financial Event Period Start Date | STRING | Mapped | `.GL_DATE` | ✓ but should be DATE type |
| Financial Event Period End Date | STRING | Mapped | `.TRAN_DATE` | ✓ but should be DATE type |
| Financial Event Covered Value | STRING | Not available | — | — |
| Financial Event COA | STRING | Not available | — | COA linkage missing — reduces GL reconciliation capability |
| Ledger Batch Id | STRING | Mapped | `TBAADM.GL_SUB_HEAD_TABLE.GL_CODE` | GL code ≠ batch ID — semantic mismatch |
| Fund Transfer Event Ind | STRING | Not available | — | — |
| Financial Event Type Code | STRING | Mapped | `.EVENT_ID` | **Critical**: mapping `Financial Event Type Code` to `EVENT_ID` — an ID is not a type code |

#### 4.14c — Metadata Completeness

All mandatory metadata absent.

**Findings:**
- F-050: Wrong SA — belongs in `silver.transaction` or `silver.gl` (High, P0)
- F-051: `Event Local Amt` and `Event Global Amt` mapped to collection/nostro amounts — semantic mismatch (High, P0)
- F-052: `Financial Event Type Code` mapped to `EVENT_ID` — type code cannot be an ID (Critical, P0)
- F-053: `Ledger Batch Id` mapped to `GL_CODE` — batch ID ≠ GL code (High, P0)
- F-054: All amounts stored as STRING (High, P0)

---

### 4.15 Entity: Fund Transfer Event

**Logical Definition:** A fund transfer event including host number, effective date, reference number, method, type, and bank transfer event type.  
**Physical Table:** `silver_dev.deposits.fund_transfer_event` (0 files, 0 bytes — empty)  
**Confidence:** 0.88

#### 4.15a — Industry Fitness

Fund transfers belong to `silver.payment` (Payments SA #13). Per `sa/subject-areas.md`, Payments SA covers "SWIFT, UAEFTS/CBUAE Clearing, ISO 8583 network data, Correspondent Banking, and Sanctions Screening." Fund transfers are a payment method, not a deposit product. The physical table is empty (0 bytes), confirming this entity has not been populated.

#### 4.15b — Attribute-Level Review

| Attribute | Type | Mapping Status | Source | Issues |
|---|---|---|---|---|
| Event Id | STRING | Mapped | `TBAADM.DC_EVENTS_TABLE.EVENT_ID` | ✓ |
| Funds Transfer Host Number | STRING | Mapped | `TBAADM.LOGIN_TABLE.HOST_NAME` | Host name from login table ≠ host number for transfer — semantic questionable |
| Fund Transfer Effective Date | STRING | Mapped | `TBAADM.MF_TRANS_DETLS_TBL.FT_FX_TRAN_DATE` | FX transaction date — should be DATE type |
| Fund Transfer Reference Number | STRING | Mapped | `TBAADM.MF_TRANS_DETLS_TBL.REF_AMT` | **Critical**: `REF_AMT` is an amount field, not a reference number |
| Fund Transfer Method Type Code | STRING | Mapped | `.TRAN_TYPE` | Raw code — SLV-004 |
| Fund Transfer Type Code | STRING | Mapped | `.PMNT_TYPE` | Raw code — SLV-004 |
| Bank Transfer Event Type Code | STRING | Mapped | `TBAADM.TRAN_CATGR_CODE_TABLE.INTER_BR_FT_TYPE` | ✓ correct source |

#### 4.15c — Metadata Completeness

All metadata absent. Table is empty.

**Findings:**
- F-055: Wrong SA — must move to `silver.payment` (Medium, P1)
- F-056: `Fund Transfer Reference Number` mapped to `REF_AMT` (amount field) — critical semantic error (High, P0)
- F-057: Table is empty — data pipeline not implemented (High, P0)

---

### 4.16 Entity: InterBank Event

**Logical Definition:** An interbank transaction event linking interbank account numbers.  
**Physical Table:** `silver_dev.deposits.interbank_event` (0 files, 0 bytes — empty)  
**Confidence:** 0.88

#### 4.16a — Industry Fitness

Interbank events belong to `silver.payment` (Payments SA). Interbank settlements are payment infrastructure (nostro/vostro) — per `sa/subject-areas.md` "Correspondent Banking" is in the Payments SA. The physical table is empty.

#### 4.16b — Attribute-Level Review

All 4 attributes have status "Not Available":

| Attribute | Type | Mapping Status | Issues |
|---|---|---|---|
| Event Id | STRING | Not Available | Source identified but data not available |
| Interbank Account Number | STRING | Not Available | Source identified but data not available |
| Interbank Account Modifier Number | STRING | Not Available | No source |
| InterBank Counter Account Number | STRING | Not Available | No source |

0% mapping coverage. This entity has no data and no viable source mappings.

#### 4.16c — Metadata Completeness

All metadata absent.

**Findings:**
- F-058: Wrong SA (Medium, P1)
- F-059: 0% mapping coverage — entity is unfunded (High, P1)
- F-060: Physical table empty (High, P1)

---

### 4.17 Entity: Reference_ATM

**Logical Definition:** ATM channel reference data including ATM ID, serial number, software release, model, and network cost.  
**Physical Table:** None found  
**Confidence:** 0.90

#### 4.17a — Industry Fitness

ATM reference data belongs in `silver.org` (Organisation SA) — ATMs are physical assets of the bank. Per `sa/subject-areas.md`, Organisation SA covers "ATMs, and Physical Assets". The underscore in `Reference_ATM` entity name also violates NAM-001 (should be `reference_atm` or `atm` depending on table vs entity name convention).

#### 4.17b — Attribute-Level Review

| Attribute | Type | Mapping Status | Source | Issues |
|---|---|---|---|---|
| Channel Id | STRING | Mapped | `TBAADM.ATM_MAST_TABLE.ATM_MNEMONIC` | ATM mnemonic ≠ channel ID |
| Channel Type Code | STRING | Mapped | `CUSTOM.TD_ACCOUNTS.CHANNEL_ID` | Source is TD accounts, not ATM master — wrong source |
| Model Id | STRING | Mapped | `ATM_MAST_TABLE.ATM_MNEMONIC` | Mnemonic used for both channel and model ID — data quality risk |
| ATM Id | STRING | Mapped | `TBAADM.ATM_CONTROLLER_TBL.ATM_CNTRL_ID` | ✓ correct source |
| ATM Software Release Num | STRING | Not Available | — | — |
| ATM Serial Num | STRING | Mapped | `TBAADM.COLTRL_MCH_DT_TABLE.MACHINE_NUM` | ✓ |
| Network Unit Cost Amt | STRING | Not Available | — | **SLV-010** would apply: amount as STRING |
| Fixed Asset Id | STRING | Not Available | — | Typo in source: `Fixed Assest Id` → `Fixed Asset Id` |

#### 4.17c — Metadata Completeness

All metadata absent. No physical table.

**Findings:**
- F-061: Wrong SA — move to `silver.org` (Medium, P1)
- F-062: `ATM_MNEMONIC` used for both `Channel Id` and `Model Id` — data duplication/confusion (Medium, P1)
- F-063: Entity name `Reference_ATM` uses PascalCase with underscore — NAM-001 violation (Low, P2)
- F-064: Attribute name typo `Fixed Assest Id` → `Fixed Asset Id` (Low, P2)

---

### 4.18 Entity: Reference_Currency

**Logical Definition:** ISO currency reference including code, numeric ID, description, country, and minor unit.  
**Physical Table:** `silver_dev.deposits.reference_currency` (1 file, 7.7 KB)  
**Confidence:** 0.98

#### 4.18a — Industry Fitness

ISO currency reference data belongs in `silver.reference` (Reference Data SA #1 — Priority Critical). Per `sa/subject-areas.md`, Reference Data SA covers "ISO Currencies, Country Codes, FX Rates". This entity is definitively misplaced. Furthermore:
- There is a **duplicate**: `reference_currency_code` is a second currency reference table in the same schema, with overlapping content.
- The physical table has 0 mapping coverage in the logical model (all `Mapping Available: None`).

#### 4.18b — Attribute-Level Review

| Attribute | Type | Mapping Status | Issues |
|---|---|---|---|
| Currency code | STRING | None | No mapping in logical model — relies on physical implementation |
| Numeric Id | STRING | None | Should be INTEGER or STRING per ISO 4217 |
| Currency description | STRING | None | — |
| Country | STRING | None | Should reference `country_code` (ISO 3166) as FK |
| Minor Unit | STRING | None | Should be INTEGER (e.g., 2 for AED) |
| Source System Code–Deleted Date | STRING | None | Standard audit columns |

No attributes are mapped in the data mapping workbook (the `Reference_Currency_Code` entity in the mapping covers 4 attributes from `CRMUSER.CURRENCYMASTER` — a different physical table).

#### 4.18c — Metadata Completeness

All metadata absent.

**Findings:**
- F-065: Wrong SA — must move to `silver.reference` (High, P0)
- F-066: Duplicate with `reference_currency_code` — consolidation required (Medium, P1)
- F-067: `Minor Unit` should be INTEGER not STRING (Low, P2)

---

### 4.19 Entity: Reference_Exchange_Rate

**Logical Definition:** FX rate between two currencies on a given date.  
**Physical Table:** `silver_dev.deposits.reference_exchange_rate` (2 files, 9.9 KB)  
**Confidence:** 0.98

#### 4.19a — Industry Fitness

FX rates belong in `silver.reference` (Reference Data SA). This is explicitly stated in `sa/subject-areas.md`: "FX Rates". Furthermore, the logical model contains **duplicate attributes**:
- `Source to Target Currency Rate__3466` and `Target to Source Currency Rate__3467` — these are duplicates of `Source to Target Currency Rate` and `Target to Source Currency Rate` with suffix numbering, indicating a modeling tool export artifact (probably Erwin duplicate detection).

#### 4.19b — Attribute-Level Review

| Attribute | Type | Mapping Status | Source | Issues |
|---|---|---|---|---|
| Source Currency Code | STRING | Mapped | `TBAADM.CRNCY_TO_CRNCY_TABLE.TRAD_CRNCY_CODE_1` | ✓ |
| Target Currency Code | STRING | Mapped | `.REVAL_CRNCY_CODE` | ✓ |
| Exchange Rate Date | STRING | Mapped | `.LCHG_TIME` | **Semantic mismatch**: LCHG_TIME is the last change timestamp, not the exchange rate effective date |
| Source to Target Currency Rate | STRING | Mapped | `.REVAL_FUTURE_RATE_CODE` | Rate stored as STRING; should be DECIMAL(18,8) for precision |
| Target to Source Currency Rate | STRING | Mapped | `.REVAL_RATE_CODE` | Same issue; also "RATE_CODE" suggests this is a code not a rate |
| Source to Target Currency Rate__3466 | STRING | None | — | Duplicate attribute — Erwin export artifact |
| Target to Source Currency Rate__3467 | STRING | None | — | Duplicate attribute — Erwin export artifact |

#### 4.19c — Metadata Completeness

All metadata absent.

**Findings:**
- F-068: Wrong SA (High, P0)
- F-069: `Exchange Rate Date` mapped to `LCHG_TIME` (last change time ≠ rate date) (High, P0)
- F-070: Duplicate attributes `__3466`/`__3467` — Erwin artifact (Medium, P1)
- F-071: Rates stored as STRING; should be DECIMAL(18,8) (Medium, P1)

---

### 4.20 Entity: Reference_Transaction_Code

**Logical Definition:** Transaction code reference table with type, description, processing type, reversal flag, settlement type, risk level, fees, and amounts.  
**Physical Table:** `silver_dev.deposits.reference_transaction_code` (584 files, 3 MB)  
**Confidence:** 0.90

#### 4.20a — Industry Fitness

Transaction code reference data belongs in `silver.reference` (Reference Data SA). While deposit-specific transaction codes could justifiably be in the Deposit SA, the entity as modeled is a **generic transaction code lookup** — it includes `Account type`, `Electronic Fund Transfer Indicator`, `Risk level`, `Processing type` — covering all transaction types across the bank, not just deposits. This confirms Reference Data SA ownership.

Additionally, the entity contains **transactional amount attributes** (`Transaction Amount`, `Transaction Local Amount`, `Transaction Global Amount`, `Transaction Fee`, `Transaction amount limit`) which are:
1. Violating SLV-006/SLV-007 (derived metrics in a reference entity).
2. Typed as STRING — violating SLV-010.
3. Semantically wrong for a code reference table (a code table should define codes, not carry transaction amounts).

#### 4.20b — Attribute-Level Review

| Attribute | Type | Mapping Status | Source | Issues |
|---|---|---|---|---|
| Transaction code | STRING | Mapped | `TBAADM.HIST_TRAN_DTL_TABLE.TRAN_ID` | **Critical**: TRAN_ID is a transaction ID, not a transaction code |
| Transaction type | STRING | None (not in mapping) | — | Missing from mapping |
| Transaction code description | STRING | None | — | Missing |
| Processing type | STRING | None | — | Missing |
| Reversal allowed | STRING | None | — | Should be BOOLEAN |
| Settlement type | STRING | None | — | Missing |
| Risk level | STRING | None | — | Risk level on a code table is unusual |
| Currency code | STRING | None | — | Missing |
| Status | STRING | None | — | Should be `transaction_code_status_code` (NAM-001) |
| Debit/Credit indicator | STRING | None | — | Column name uses `/` — invalid for column naming (NAM-001) |
| Fee applicable | STRING | None | — | Should be BOOLEAN `is_fee_applicable` |
| Account type | STRING | Mapped | `CRMUSER.ACCOUNTS.ACCOUNTTYPE` | — |
| Transaction amount limit | STRING | None | — | **SLV-007**: Derived limit in reference table |
| Transaction Local Amount | STRING | Mapped | `TBAADM.HIST_TRAN_DTL_TABLE.REF_AMT` | **SLV-006/007**: Transaction amounts in a code reference table |
| Transaction Global Amount | STRING | Mapped | `.TOT_OUT_AMOUNT` | Same violation |
| Transaction Fee | STRING | Mapped | `TBAADM.MULTI_CRNCY_TRAN_DET_TBL.TRAN_RATE` | TRAN_RATE ≠ transaction fee |
| Electronic Fund Transfer Indicator | STRING | Mapped | `TBAADM.TD_TRAN_TABLE.PART_TRAN_TYPE` | PART_TRAN_TYPE is debit/credit part — not EFT indicator |
| Ledger Batch Id | STRING | Mapped | `TBAADM.DAILY_TRAN_DETAIL_TABLE.GL_SUB_HEAD_CODE` | GL sub-head code ≠ ledger batch ID |
| Transaction Start/End Date | STRING | Mapped | TRAN_DATE / PSTD_DATE | Dates on a code table are unusual; stored as STRING |
| Payment Event Ind | STRING | Mapped | `.TRAN_TYPE` | TRAN_TYPE as payment event indicator is ambiguous |

#### 4.20c — Metadata Completeness

All metadata absent.

**Findings:**
- F-072: Wrong SA — move to `silver.reference` (High, P0)
- F-073: `Transaction code` mapped to `TRAN_ID` — ID ≠ code (Critical, P0)
- F-074: Transaction amounts in a code reference table — SLV-006/SLV-007 violation (High, P0)
- F-075: `Debit/Credit indicator` column name uses `/` — NAM-001 violation (Low, P2)
- F-076: `Fee applicable` and `Reversal allowed` should be BOOLEAN (Low, P2)

---

### 4.21 Entity: Safe Deposit Account

**Logical Definition:** A safe deposit box account — vault rental service for secure storage.  
**Physical Table:** `silver_dev.deposits.safe_deposit_account` (5 files, 31 MB)  
**Confidence:** 0.85

#### 4.21a — Industry Fitness

Safe deposit box services are within the scope of the Deposit SA (BIAN service domain: **"Safe Custody"** SD-104, or as an extension of **"Deposit Account Facility"**). However, the entity is severely under-modeled:
- No box location
- No rental fee
- No contract/lease period
- No key information
- No customer/party FK

#### 4.21b — Attribute-Level Review

| Attribute | Type | Mapping Status | Source | Issues |
|---|---|---|---|---|
| Account Number | STRING | Mapped | `TBAADM.GENERAL_ACCT_MAST_TABLE.FORACID` | BK candidate — should become `safe_deposit_account_bk` |
| Account Modifier Number | STRING | Mapped | `TBAADM.GENERAL_ACCT_MAST_TABLE.ACID` | Internal key |
| Deposit Box Size Code | STRING | Not Available | None | Critical attribute — box size determines rental fee |
| Safe Deposit Box Number | STRING | Not Available | None | Critical identifier — the box number is the primary business key |

2 of 4 attributes are "Not Available". The primary business key (`Safe Deposit Box Number`) has no source mapping.

#### 4.21c — Metadata Completeness

All mandatory metadata absent.

**Findings:**
- F-077: Primary business key (`Safe Deposit Box Number`) has no source mapping (High, P0)
- F-078: Missing customer FK, rental period, location (Medium, P1)
- F-079: All SK/BK/SCD-2 missing (High, P0)

---

### 4.22 Entity: Term Deposit Transaction

**Logical Definition:** Transaction records for term deposit accounts including amount, exchange rate, account details, and channel information.  
**Physical Table:** `silver_dev.deposits.term_deposit_transaction` (916,617 files, 7.6 GB)  
**Confidence:** 0.88

#### 4.22a — Industry Fitness

Transaction data is the domain of `silver.transaction` (Transaction SA #12): "Atomic Double-Entry Postings, Card Settlements, POS Transactions, Fee Reversals, and Interest Capitalization." Placing term deposit transactions in the Deposit SA means:
1. The Transaction SA's fact tables cannot provide a complete transaction view without joining the Deposit SA (cross-SA join — violates §2.2).
2. Term deposit interest capitalization, rollovers, and early redemption penalties are not in `silver.transaction`, creating gaps in the financial ledger.
3. 916,617 files — severe file fragmentation; typical for high-frequency transaction tables without compaction.

The entity is the second-largest in the schema (7.6 GB) and represents real production data. However, its SA placement is architecturally wrong.

**Recommendation**: Move `term_deposit_transaction` to `silver.transaction` as `term_deposit_transaction` (or merge with the generic transaction table using a type discriminator).

#### 4.22b — Attribute-Level Review

| Attribute | Type (Logical) | Mapping Status | Source | Issues |
|---|---|---|---|---|
| Transaction Id | STRING | Mapped | `TBAADM.HIST_TRAN_DTL_TABLE.TRAN_ID` | BK candidate — should become `term_deposit_transaction_bk` |
| Currency code | STRING | None (not in mapping doc) | — | Missing from mapping |
| Transaction type | STRING | None | — | Missing; raw type code would violate SLV-004 |
| Transaction code | STRING | Mapped | `CUSTOM.HIST_TRAN_DTL_TABLE.TRAN_TYPE` | Raw code — SLV-004 |
| Account Number | STRING | Mapped | `TBAADM.general_acct_mast_table.ACCT_NUM` | FK to deposit_account — should be `deposit_account_bk` |
| Account Type Account Currency | STRING | Mapped | `TBAADM.HIST_TRAN_DTL_TABLE.TRAN_CRNCY_CODE` | Attribute name conflates two concepts (account type + currency) |
| Transaction Date | STRING | Mapped | `HIST_TRAN_DTL_TABLE.TRAN_DATE` | Should be DATE type |
| Transaction Time | STRING | Mapped | `HIST_TRAN_DTL_TABLE.TRAN_DATE` | Transform: `TIME(TRAN_DATE)` — splitting date/time from same column |
| Credit Or Debit Indicator | STRING | Mapped | `TD_TRAN_TABLE.PART_TRAN_TYPE` | Raw code — SLV-004; should be BOOLEAN or canonical 'CR'/'DR' |
| Transaction Affect Code | STRING | Not Available | — | No source |
| Transaction Amount | STRING | Mapped | `HIST_TRAN_DTL_TABLE.TRAN_AMT` | **SLV-010**: Amount as STRING |
| Exchange Rate | STRING | Mapped | `PAYMENT_ORDER_DETAIL_TABLE.ORIG_EXCH_RATE` | Should be DECIMAL(18,8) |
| Base Currency Transaction Amount | STRING | Mapped | `HIST_TRAN_DTL_TABLE.TRAN_AMT` | Same source as `Transaction Amount` — duplicate |
| Transaction Currency | STRING | Mapped | `Hist_Tran_Dtl_Table.TRAN_CRNCY_CODE` | Duplicate of `Account Type Account Currency` source |
| Input Source | STRING | Not Available | — | — |
| Sequence Id | STRING | Mapped | None.None.None.None | Listed as Mapped but source is null — contradiction |
| Posting Sequence Id | STRING | Mapped | None.None.None.None | Same issue |
| Effective Account Number | STRING | Mapped | `general_acct_mast_table.ACCT_NUM` | Duplicate of Account Number source |
| Effective Account Type | STRING | Mapped | `TD_ACCT_MASTER_TABLE.DEPOSIT_TYPE` | ✓ |
| Effective Account Currency | STRING | Mapped | Complex multi-join CTE (`crmuser.salebackend`) | Very complex transform for a simple reference — DQ risk |
| Auxiliary Transaction Code | STRING | Mapped | `TD_TRAN_TABLE.TRAN_ID` | Tran ID mapped as aux code — semantic error |
| Auxiliary Transaction Code Description | STRING | Mapped | `TD_TRAN_TABLE.TRAN_RMKS` | ✓ (remarks as description) |
| Transaction Your Reference | STRING | Mapped | `CUSTOM.TD_ACCOUNTS.REF_NUM` | Both "Your" and "Our" reference map to same `REF_NUM` |
| Transaction Our Reference | STRING | Mapped | `CUSTOM.TD_ACCOUNTS.REF_NUM` | Duplicate source — meaningless distinction |
| Transaction Text | STRING | Mapped | `TD_TRAN_TABLE.TRAN_RMKS` | ✓ |
| Transaction Channel | STRING | Mapped | `CUSTOM.TD_ACCOUNTS.CHANNEL_ID` | Raw channel ID — no canonical mapping |
| Transaction User | STRING | Mapped | `TD_ACCT_MASTER_TABLE.LCHG_USER_ID` | Last change user ≠ transaction user |
| Transaction Account Branch | STRING | Mapped | `TD_ACCT_MASTER_TABLE.BANK_ID` | Bank ID from account master — should be branch code |
| Transaction Source Branch | STRING | Not Available | — | No source |
| Customer Number | STRING | Mapped | `CUSTOM.TD_ACCOUNTS.CUST_ID` | Should be FK to `silver.party`; stored as raw customer ID |
| Segment | STRING | Not Available | — | Customer segment in a transaction entity — denormalization risk |
| Segment Description | STRING | Not Available | — | Same denormalization concern |
| Source System Code | STRING | Mapped | None.None.None.None | Listed as Mapped but source is null |
| Source System Id | STRING | Mapped | None.None.None.None | Listed as Mapped but source is null |
| Business Start Date | CHAR(18) | — | — | SCD-2 proxy; wrong type |
| Business End Date | CHAR(18) | — | — | SCD-2 proxy; wrong type |
| Is Active Flag | CHAR(18) | — | — | Wrong type |

#### 4.22c — Metadata Completeness

Partial — `Source System Code`, `Source System Id`, `Business Start Date`, `Business End Date`, `Is Active Flag` are present but typed incorrectly. The full set of 11 mandatory metadata columns is not complete.

**Findings:**
- F-080: SA boundary violation — transactions belong in `silver.transaction` (High, P0)
- F-081: 916K files — critical fragmentation; requires OPTIMIZE (High, P0)
- F-082: All monetary amounts (`Transaction Amount`, `Base Currency Transaction Amount`) as STRING (High, P0)
- F-083: `Auxiliary Transaction Code` mapped to `TRAN_ID` — semantic error (High, P0)
- F-084: `Sequence Id` and `Posting Sequence Id` listed as Mapped but source is null (Medium, P1)
- F-085: `Transaction Your Reference` and `Transaction Our Reference` mapped to same source (Medium, P1)
- F-086: `Segment` and `Segment Description` in a transaction entity — unnecessary denormalization (Medium, P1)
- F-087: `Transaction User` mapped to last-change user, not transaction initiating user (Medium, P1)
- F-088: `Customer Number` stored as raw ID — should be FK to `silver.party` (Medium, P1)
- F-089: `Credit Or Debit Indicator` stores raw `PART_TRAN_TYPE` — SLV-004 violation (Medium, P1)

---

## 5. Denormalization Register

| # | Co-location | Entity | Classification | Justification Required | Recommendation |
|---|---|---|---|---|---|
| D-001 | `Customer Number` in `Term Deposit Transaction` | term_deposit_transaction | **Unnecessary** | No — account number is sufficient FK; customer resolves via Account SA | Remove; use `account_bk` → resolve customer in Gold via account-party relationship |
| D-002 | `Segment` + `Segment Description` in `Term Deposit Transaction` | term_deposit_transaction | **Unnecessary** | No — segment is a Party SA attribute | Remove; join at Gold layer |
| D-003 | `Transaction Code Description` in `Term Deposit Transaction` | term_deposit_transaction | **Acceptable** | Yes — commonly denormalized for operational reporting convenience | Retain only if the reference code table has join performance issues; document steward approval |
| D-004 | `Auxiliary Transaction Code Description` in `Term Deposit Transaction` | term_deposit_transaction | **Acceptable** | Yes — same rationale as D-003 | Same recommendation |
| D-005 | `Account Type Account Currency` conflating two attributes | term_deposit_transaction | **Unnecessary** | No — account type and account currency are separate concepts | Split into `account_type_code` and `account_currency_code` |
| D-006 | `Acct Currency Original Deposit Amt` naming (currency code in amount entity) | deposit_account | **Unnecessary** | No — currency code should be a separate attribute | Rename to `original_deposit_currency_code` and `original_deposit_amount_orig` |
| D-007 | `Transaction Local Amount` + `Transaction Global Amount` in `Reference_Transaction_Code` | reference_transaction_code | **Unnecessary** | No — amounts do not belong in a code reference table | Remove; these are transactional facts |
| D-008 | `Financial Agreement` attributes across `Deposit_Account` and separate entity | Financial_Agreement | **Unnecessary** | No — Financial_Agreement is an orphaned mapping entity not in logical model | Align mapping to logical model or create proper `deposit_agreement` entity |

---

## 6. Guideline Compliance Summary

### 6.1 Rules Assessment Table

| Rule | Rule Text (Abbreviated) | Status | Finding IDs | Priority |
|---|---|---|---|---|
| **SLV-001** | Every entity must have `<entity>_sk` MD5 surrogate key | 🔴 Violated — 0/22 entities | F-004, F-012, F-025, F-039, F-043, F-047, F-079 (all) | P0 |
| **SLV-002** | Every entity must have `<entity>_bk` business key | 🔴 Violated — 0/22 entities | All entity findings | P0 |
| **SLV-003** | SCD-2 default: `effective_from`, `effective_to`, `is_current` | 🔴 Violated — `Business Start/End Date` (CHAR(18)) is non-standard proxy | F-004, F-027, F-039, F-043, F-047, F-079 | P0 |
| **SLV-004** | No source-system codes; use `silver.reference.code_mapping` | 🔴 Violated — multiple raw codes stored: `TRAN_TYPE`, `CHANNEL_LEVEL_CODE`, `PART_TRAN_TYPE` | F-008, F-053, F-075, F-089 | P1 |
| **SLV-005** | DQ-gated writes; quarantine table | 🟡 Unknown — no quarantine tables visible in physical schema | — | P1 |
| **SLV-006** | 3NF; no derived values/aggregations | 🔴 Violated — `reference_transaction_code` contains transaction amounts; `deposit_balance_report` is a report | F-074, via ARCH-001 | P0 |
| **SLV-007** | No pre-computed metrics | 🔴 Violated — `deposit_balance_report` is a balance report; `reference_transaction_code` has amount limits | F-074, ARCH-001 | P0 |
| **SLV-008** | All mandatory metadata columns present | 🔴 Violated — all 22 entities missing some or all of the 11 mandatory columns | All entity 4.xc findings | P0 |
| **SLV-009** | All timestamps UTC | 🟡 Unknown — all dates stored as STRING; UTC enforcement cannot be verified | F-005, F-038, F-044 | P0 |
| **SLV-010** | Monetary amounts in smallest unit (fils for AED) | 🔴 Violated — all amounts stored as STRING throughout: `Original Deposit Amt`, `Deposit Slip Amt`, `Transaction Amount`, `Event Amt`, etc. | F-028, F-037, F-054, F-082 | P0 |
| **NAM-001** | snake_case, lowercase, no reserved words | 🟡 Partially violated — `deffered_income_tax_account` (typo), `IsLogicallyDeleted`, `Reference_ATM`, `Debit/Credit indicator` | F-023, F-075, F-063 | P1 |
| **NAM-003** | Table naming: `<entity>` for Silver | 🟡 Partially violated — `deposit_account_old`, `*_temp_no_partition`, `deposits_kvp` in production schema | ARCH-001 | P1 |
| **§2.1 (3NF)** | No repeating groups, no transitive dependencies | 🟡 Partially violated — `Account Type Account Currency` conflates two attributes; `Segment`/`Segment Description` in transaction entity | D-005, D-002 | P1 |
| **§2.2 (SA Isolation)** | Subject areas self-contained | 🔴 Violated — 13+ out-of-scope entities in Deposit SA | ARCH-001 | P0 |
| **§2.4 (MD5 SK)** | MD5 deterministic surrogate keys | 🔴 Violated — no surrogate keys defined | ARCH-003 | P0 |
| **§2.5 (Code Canonicalization)** | Canonical codes from code_mapping | 🔴 Violated — raw CBS codes stored throughout | SLV-004 findings | P1 |
| **§2.6 (No Aggregations)** | No aggregations in Silver | 🔴 Violated — `deposit_balance_report` and amounts in `reference_transaction_code` | F-074 | P0 |
| **§2.7 (Audit Columns)** | 11 mandatory Silver audit columns | 🔴 Violated — no entity has all 11 columns in logical model | All 4.xc findings | P0 |
| **§2.9 (UTC)** | UTC timestamps only | 🔴 Cannot verify — all timestamps stored as STRING | SLV-009 findings | P0 |
| **§2.11 (Erwin Model)** | Model before DDL | 🟡 Partial — logical model exists but physical tables have no corresponding Erwin entity for `deposits_kvp`, `deposit_account_old`, temp tables | ARCH-001 | P1 |
| **§2.12 (Catalog)** | Register in Informatica GDGC | 🔴 Violated — most physical tables have `description: null` in Unity Catalog metadata | ARCH-001 | P1 |
| **§Gate 2 (DECIMAL)** | Monetary columns `DECIMAL(18,4)` | 🔴 Violated — all amounts as STRING | SLV-010 findings | P0 |
| **§Gate 2 (PII Masking)** | EID, Card Numbers, Phone, Passport masked via SHA-256 | 🟡 Unknown — `Payor Check Account Number` not confirmed masked | F-011 | P0 |

### 6.2 Compliance Score Summary

| Category | Rules Checked | Compliant | Partial | Violated | Unknown |
|---|---|---|---|---|---|
| Identity / Key Strategy | 2 | 0 | 0 | 2 | 0 |
| SCD / History | 1 | 0 | 0 | 1 | 0 |
| Monetary Types | 1 | 0 | 0 | 1 | 0 |
| SA Isolation / Scoping | 2 | 0 | 0 | 2 | 0 |
| Metadata Completeness | 2 | 0 | 0 | 2 | 0 |
| Code Canonicalization | 2 | 0 | 1 | 1 | 0 |
| 3NF / Aggregations | 3 | 0 | 1 | 2 | 0 |
| Naming | 2 | 0 | 2 | 0 | 0 |
| DQ / PII | 3 | 0 | 0 | 1 | 2 |
| Catalog / Governance | 2 | 0 | 1 | 1 | 0 |
| **TOTAL** | **20** | **0** | **5** | **13** | **2** |

**Overall Compliance Rate: 0/20 fully compliant (0%). 13/20 rules actively violated.**

---

## 7. Remediation Plan

### 7.1 Prioritized Actions

#### P0 — Immediate (Blocking Production Sign-Off)

| # | Action | Entities Affected | Effort | Owner | Guideline |
|---|---|---|---|---|---|
| R-001 | Define `<entity>_sk` (MD5) and `<entity>_bk` on every in-scope entity in Erwin logical model; generate DDL; backfill physical tables via pipeline re-run | All in-scope: `deposit_account`, `deposit_term_account`, `deposit_transaction_account`, `deposit_slip`, `safe_deposit_account` | 6–8 weeks | Data Modeler + ETL Engineer | SLV-001, SLV-002 |
| R-002 | Add SCD-2 columns (`effective_from TIMESTAMP`, `effective_to TIMESTAMP`, `is_current BOOLEAN`) to all in-scope entities; deprecate `Business Start Date` / `Business End Date` CHAR(18) | All in-scope entities | 4–6 weeks | Data Modeler + ETL Engineer | SLV-003 |
| R-003 | Migrate all monetary columns from STRING to `DECIMAL(18,4)` or `BIGINT` (for fils); update all pipelines | `deposit_account.original_deposit_amt`, `deposit_slip.deposit_slip_amt`, `term_deposit_transaction.transaction_amount`, `financial_event.*_amt` | 3–4 weeks | ETL Engineer | SLV-010, §Gate 2 |
| R-004 | Relocate out-of-scope entities to correct SAs: Collateral items → `silver.collateral`; GL entities → `silver.gl`; Payments events → `silver.payment`; Reference tables → `silver.reference`; Channel → `silver.org` | 14 tables | 2–3 weeks | Data Architect + multiple team owners | §2.2 |
| R-005 | Fix critical semantic mapping errors: (a) `Account Modifier Number` ≠ `MODIFIED_USERID`; (b) `Grace Period End Date` ≠ `NUM_OF_GRACE_DAYS_UTIL`; (c) `Transaction Code` ≠ `TRAN_ID`; (d) `Fund Transfer Reference Number` ≠ `REF_AMT`; (e) `Financial Event Type Code` ≠ `EVENT_ID` | Multiple entities | 1–2 weeks | ETL Engineer + Business Analyst | SLV-006, §2.4 |
| R-006 | Add all 11 mandatory Silver audit columns to every entity logical model and physical DDL | All 22 entities | 3–4 weeks | Data Modeler | §2.7, SLV-008 |
| R-007 | Remove `deposit_balance_report` from Silver schema (violates SLV-006/SLV-007); if needed, move to Gold as `fact_deposit_balance_daily` | `deposit_balance_report` | 1 day | Data Architect | SLV-006, SLV-007 |
| R-008 | Fix PK gap: `deposit_slip.document_id` and `safe_deposit_account.safe_deposit_box_number` have no source mapping; identify source or defer entity | `deposit_slip`, `safe_deposit_account` | 2–3 weeks | Business Analyst + Source SME | SLV-001 |

#### P1 — Next Sprint (Quality Improvements)

| # | Action | Entities Affected | Effort | Owner | Guideline |
|---|---|---|---|---|---|
| R-009 | Remove/archive temp and legacy tables: `deposit_account_old`, `*_temp_no_partition` (×3), `deposits_kvp` | Physical schema | 2 days | Platform Engineer | NAM-003, §2.11 |
| R-010 | Rename `deffered_income_tax_account` → `deferred_income_tax_account` (NAM-001 typo fix) and then relocate to `silver.gl` | 1 table | 1 day | Data Modeler | NAM-001 |
| R-011 | Consolidate `reference_currency` and `reference_currency_code` into a single canonical entity in `silver.reference` | 2 tables | 1 week | Reference Data Owner | §2.2 |
| R-012 | Resolve duplicate attributes in `Reference_Exchange_Rate` (`__3466`, `__3467`) — remove Erwin export artifacts from logical model | `Reference_Exchange_Rate` | 1 day | Data Modeler | §2.11 |
| R-013 | Add FK attributes to `deposit_account`: `party_bk` (→ `silver.party`), `account_bk` (→ `silver.account`), `product_bk` (→ `silver.product`) | `deposit_account` | 1–2 weeks | Data Modeler + ETL | ARCH-005 |
| R-014 | Replace `IsLogicallyDeleted` with `is_active_flag` (STRING 'Y'/'N') across all entities | All entities | 1 week | Data Modeler | NAM-001, §2.7 |
| R-015 | Implement canonical code mapping for raw source codes via `silver.reference.code_mapping`: `TRAN_TYPE`, `PART_TRAN_TYPE`, `CHANNEL_LEVEL_CODE` | `term_deposit_transaction`, `bofd_check_event`, `financial_event` | 2–3 weeks | ETL Engineer + Reference Data Owner | SLV-004 |
| R-016 | Convert all date/timestamp STRING fields to proper DATE or TIMESTAMP type with UTC enforcement | All entities with string dates | 2–3 weeks | ETL Engineer | SLV-009 |
| R-017 | Confirm PII masking for `Payor Check Account Number` in `bofd_check_image` | `bofd_check_image` | 1 day | Data Security / Platform | §Gate 2, §10 |
| R-018 | Register all Silver tables in Unity Catalog / Informatica GDGC with description, steward, and lineage | All 26 tables | 1 week | Data Steward | §2.12 |
| R-019 | Run `OPTIMIZE` + `ZORDER` on `deposit_account` (4.1M files) and `term_deposit_transaction` (917K files) | 2 tables | 2 days (platform ops) | Platform Engineer | §3.7 |

#### P2 — Backlog (Model Enrichment)

| # | Action | Effort | Guideline |
|---|---|---|---|
| R-020 | Add missing BIAN-aligned entities: `deposit_interest_schedule`, `deposit_rollover_instruction`, `deposit_early_redemption` | 4–6 weeks | ARCH-002 |
| R-021 | Separate CASA and Term Deposit into distinct entity hierarchies | 4–6 weeks | ARCH-002 |
| R-022 | Add Islamic product attributes: `is_islamic_product`, `profit_rate`, `sharia_structure_code` | 2–3 weeks | ARCH-002 |
| R-023 | Evaluate whether `deposit_transaction_account` is redundant (bridge table) and remove if confirmed | 1 week | F-045 |
| R-024 | Populate `InterBank Event` with source data or remove from schema | 2 weeks | F-059 |
| R-025 | Split `Account Type Account Currency` into two separate attributes | 1 week | D-005 |
| R-026 | Remove `Segment` and `Segment Description` from `term_deposit_transaction` (denormalization) | 1 week | D-002 |

### 7.2 Remediation Schedule

```
Week 1–2:   R-009, R-010, R-007, R-017, R-019 (quick wins + critical removes)
Week 3–6:   R-001 (SK/BK addition — most impactful)
Week 7–10:  R-002 (SCD-2 columns)
Week 11–13: R-003 (monetary type migration)
Week 14–15: R-004 (SA entity relocation)
Week 16–17: R-005 (semantic mapping fixes)
Week 18–19: R-006 (complete metadata columns)
Week 20–21: R-008 (PK source mapping)
Week 22+:   P1 items R-009 to R-019 (parallel track)
Backlog:    P2 items R-020 to R-026
```

---

## 8. Appendix

### Appendix A: Mapping Summary

| Entity | Total Logical Attrs | Mapped | Not Available | Not in Mapping | Mapping % |
|---|---|---|---|---|---|
| Account Event | 10 | 4 | 0 | 6 | 40% |
| BOFD Check Event | 9 | 2 | 1 | 6 | 22% |
| BOFD Check Image | 10 | 3 | 1 | 6 | 30% |
| Collateral Item | 9 | 1 | 2 | 6 | 11% |
| Collateral Item Class Association | 11 | 3 | 2 | 6 | 27% |
| Contact Event | 15 | 4 | 5 | 6 | 27% |
| Deffered Income Tax Account | 7 | 1 | 0 | 6 | 14% |
| Deposit Account | 17 | 5 | 2 | 10 | 29% |
| Deposit GL Account | 7 | 1 | 0 | 6 | 14% |
| Deposit Slip | 18 | 6 | 2 | 10 | 33% |
| Deposit Term Account | 13 | 4 | 0 | 9 | 31% |
| Deposit Transaction Account | 8 | 2 | 0 | 6 | 25% |
| Electronic Channel | 2 | 1 | 1 | 0 | 50% |
| Financial Event | 12 | 9 | 3 | 0 | 75% |
| Fund Transfer Event | 7 | 7 | 0 | 0 | 100% |
| InterBank Event | 4 | 0 | 4 | 0 | 0% |
| Reference_ATM | 14 | 5 | 3 | 6 | 36% |
| Reference_Currency | 11 | 0 | 0 | 11 | 0% |
| Reference_Exchange_Rate | 13 | 5 | 0 | 8 | 38% |
| Reference_Transaction_Code | 24 | 11 | 0 | 13 | 46% |
| Safe Deposit Account | 4 | 2 | 2 | 0 | 50% |
| Term Deposit Transaction | 37 | 28 | 5 | 4 | 76% |
| **TOTAL** | **252** | **104** | **33** | **115** | **41%** |

**Additional orphaned mapping entities (in mapping but not in logical model):**
- `Financial_Agreement` (9 attrs, 8 mapped)
- `Bank Event` (1 attr, 1 mapped)
- `Reference_Channel` (7 attrs, 5 mapped)
- `Reference_Currency_Code` (4 attrs, 4 mapped)
- `ATM_Channel` (8 attrs, 5 mapped)

---

### Appendix B: Guideline Citations

| Rule Code | Guideline File | Full Rule Text |
|---|---|---|
| SLV-001 | `guidelines/11-modeling-dos-and-donts.md §2.4` | "Derive surrogate keys deterministically: `MD5(UPPER(TRIM(business_key)))` for single-component keys; `MD5(CONCAT_WS('|', key_part_1, key_part_2))` for composite keys." |
| SLV-002 | `guidelines/11-modeling-dos-and-donts.md §2.4` | Column naming convention: `<entity>_bk — business/natural key` |
| SLV-003 | `guidelines/11-modeling-dos-and-donts.md §2.3` | "Track all changes to tracked attributes with SCD Type 2: add a new row with updated values, close the prior row (`effective_to` = change timestamp, `is_current` = FALSE). Every Silver entity table must have `effective_from`, `effective_to`, `is_current`." |
| SLV-004 | `guidelines/11-modeling-dos-and-donts.md §2.5` | "Resolve all source-system codes to canonical platform values using a central code-mapping reference (e.g., `silver.reference.code_mapping`). Store the canonical code, not the raw source code, in entity columns." |
| SLV-005 | `guidelines/01-data-quality-controls.md §3.2` | "Failed records routed to quarantine / error tables rather than discarded." |
| SLV-006 | `guidelines/11-modeling-dos-and-donts.md §2.6` | "Store only atomic, un-aggregated facts in Silver. DON'T pre-compute totals, averages, counts, ratios, or any derived metrics in Silver tables." |
| SLV-007 | `guidelines/01-data-quality-controls.md §4` | "Aggregations / KPIs: Prohibited in Silver; reserved for the Gold layer." |
| SLV-008 | `guidelines/11-modeling-dos-and-donts.md §2.7` | "Include the following Silver technical audit columns on every Silver entity table: `source_system_code`, `source_system_id`, `create_date`, `update_date`, `delete_date`, `is_active_flag`, `effective_from`, `effective_to`, `is_current`, `run_id`, `source_ingestion_date`." |
| SLV-009 | `guidelines/11-modeling-dos-and-donts.md §2.9` | "Store all timestamps in UTC. Convert source-local timestamps at ingestion." |
| SLV-010 | `guidelines/01-data-quality-controls.md §Gate 2` | "All monetary columns must use `DECIMAL(18,4)`" |
| NAM-001 | `guidelines/06-naming-conventions.md §General Rules` | "Use snake_case for all identifiers (catalogs, schemas, tables, columns, views). Use lowercase only — no mixed case. No abbreviations unless they are universally understood domain terms." |
| NAM-003 | `guidelines/06-naming-conventions.md §Table Naming` | "Silver entity: `<entity>` — e.g., `customer`, `account`, `transaction`." |
| §2.2 | `guidelines/11-modeling-dos-and-donts.md §2.2` | "Keep each Silver subject area self-contained. Pipelines for one subject area read only from Bronze and from the same subject area's own Silver tables." |
| §Gate 2 DQ | `guidelines/01-data-quality-controls.md §Gate 2` | "If the error rate exceeds 5%, the pipeline must FAIL and alert the Engineering team." |
| §PII | `guidelines/01-data-quality-controls.md §Gate 2` | "EID, Card Numbers, Phone Numbers, and Passport Numbers masked via SHA-256 unless stored in a restricted container." |

---

### Appendix C: Industry References

| Reference | Domain | Relevance to Findings |
|---|---|---|
| **BIAN v11 SD-017** — Deposit Account Facility | Deposit Account | Core service domain for `deposit_account`, `deposit_term_account`; maps open/maintain/close operations |
| **BIAN v11 SD-104** — Safe Custody | Safe Deposit | Service domain for `safe_deposit_account` |
| **BIAN v11 SD-088** — Customer Contact | Contact Event | Confirms Contact Event belongs in `silver.event`, not `silver.deposit` |
| **BIAN v11 SD-024** — Collateral Asset Administration | Collateral Item | Confirms Collateral entities belong in `silver.collateral` |
| **IFW (IBM Banking Framework)** — Account supertype hierarchy | Deposit Account | IFW separates CASA Account and Term Deposit Account as distinct subtypes; supports finding ARCH-002 |
| **BCBS 239** — Principle 6 (Completeness) | SK/BK/FK gaps | Requires complete, accurate risk data aggregation; missing SKs and broken FKs directly impede BCBS 239 compliance |
| **IFRS 9** — Expected Credit Loss (ECL) | N/A for core deposit | IFRS 9 applies to lending; however deposit products with embedded credit features (overdraft) require stage tracking — not present in current model |
| **CBUAE Circular 2020/2** — Deposit Protection | Safe Deposit Account | UAE requires protected deposits to be identifiable; missing account status and product type attributes impede eligibility calculation |
| **UAEFTS** — UAE Funds Transfer System | Fund Transfer Event | Confirms Fund Transfer Event belongs in `silver.payment` under UAEFTS message handling |
| **ISO 4217** | Reference_Currency | Currency codes must be ISO 4217 3-letter alpha; `Numeric Id` should align with numeric ISO codes |
| **ISO 8601** | Date/timestamp fields | All dates must conform to `YYYY-MM-DD`; all timestamps to UTC ISO 8601; currently all stored as STRING |
| **Databricks Delta Lake** — `OPTIMIZE`/`ZORDER` | File compaction | `deposit_account` (4.1M files) and `term_deposit_transaction` (917K files) require urgent compaction to prevent query degradation |

---

*End of Report*

**Document Version:** 1.0  
**Status:** Draft — Pending Data Steward Review  
**Next Review Date:** 30 days from issue  
**Distribution:** Data Platform Architect, Deposit SA Data Steward, Group Data Office
