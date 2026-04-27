---

# Silver Layer Gap Assessment — Loan Subject Area
**Assessment Date:** 2026-04-27 | **Assessor:** Senior Data Platform Architect | **Version:** 1.0

---

## 1. Executive Summary

### Management Slide

| Dimension | Rating | Summary |
|---|---|---|
| **Overall Health** | 🔴 Critical | Multiple P0 defects blocking production readiness |
| **Physical Completeness** | 🔴 Critical | 5 of 13 logical entities have no physical table; 5 physical tables are empty |
| **Metadata Compliance** | 🔴 Critical | `loan_account_master` has zero SCD/audit metadata columns |
| **Key Strategy** | 🔴 Critical | No surrogate keys (`_sk`) or business keys (`_bk`) on any table |
| **Data Typing** | 🔴 Critical | All 598 logical attributes typed `STRING`; no monetary amounts as DECIMAL |
| **3NF Compliance** | 🔴 Critical | `Loan_Account` is a 141-attribute god-table mixing 8+ distinct concerns |
| **Naming Convention** | 🟡 Improper | Active table named `loan_account_master` (should be `loan_account`); many columns PascalCase/snake-broken |
| **Reference Data Isolation** | 🟡 Improper | 3 reference tables physically co-located in `silver.loan` instead of `silver.reference` |
| **Mapping Coverage** | 🟡 Partial | ~13 attributes in Loan_Account pending or not available; ESG attributes entirely unmapped |
| **IFRS 9 Coverage** | 🔴 Critical | ECL staging entity missing entirely from both logical model and physical structures |

### Top 5 Immediate Actions

| # | Action | Priority | Owner | Effort |
|---|---|---|---|---|
| 1 | Add all 11 mandatory Silver metadata/audit columns (`source_system_code`, `source_system_id`, `create_date`, `update_date`, `delete_date`, `is_active_flag`, `effective_from`, `effective_to`, `is_current`, `run_id`, `source_ingestion_date`) to **all** loan tables | **P0** | Data Modeling + ETL | Large |
| 2 | Add `<entity>_sk` (MD5 surrogate) and `<entity>_bk` (natural key) to every Silver loan entity | **P0** | Data Modeling + DBA | Large |
| 3 | Fix all attribute data types: monetary → `DECIMAL(18,4)`, dates → `DATE`, timestamps → `TIMESTAMP`, codes → `STRING`, counts → `INT` | **P0** | Data Modeling | Large |
| 4 | Create physical tables for the 5 missing critical entities: `loan_agreement`, `loan_instalment_schedule`, `loan_statement`, `loan_statement_line_item`, `ifrs9_ecl_staging` | **P0** | DBA + ETL | Large |
| 5 | Decompose `Loan_Account` god-table: extract tuition-loan, car-loan, ESG, collection-status, credit-scoring, and disbursement concerns into separate normalized entities | **P0** | Data Modeling | Large |

---

## 2. Subject Area Architecture Assessment

### 2.1 Scope Verification

The subject area is registered as **Loan** (`silver.loan`) under the **Lending** umbrella (`silver.lending`) per `sa/subject-areas.md` (#8a). The physical catalog uses schema `silver_dev_v2.loan`.

**Scope violation found:** Two entities — `Credit_Card_Statement` (31 attrs) and `Credit_Card_Statement_Line_Item` (26 attrs) — appear in the **Loan** logical model. These belong to the **Card** subject area (`silver.card`). Their presence in the Loan model inflates scope and will cause double-maintenance.

### 2.2 Domain Decomposition vs. BIAN

| BIAN Service Domain | Expected Entity | Present? | Gap |
|---|---|---|---|
| Loan | `loan_account` (facility contract) | Partial — exists as `loan_account_master` | Renamed, non-standard |
| Consumer Loan | `loan_term_account` (repayment terms) | ✅ Present | — |
| Loan | `loan_transaction` (postings) | ✅ Present | Scope overlap with Transaction SA |
| Loan | `loan_agreement` (legal agreement) | ❌ Missing physical | Critical |
| Loan | `loan_instalment_schedule` | ❌ Missing physical | Critical |
| Financial Accounting | `loan_provision_balance_history` | ❌ Missing physical | High |
| Financial Accounting | `loan_provision_change_history` | ❌ Missing physical | High |
| IFRS 9 | `ifrs9_ecl_staging` | ❌ Entirely absent | Critical — SA description explicitly requires it |
| Loan Disbursement | Disbursement entity | ❌ Absent | Data folded into Loan_Account |
| Loan Statement | `loan_statement`, `loan_statement_line_item` | ❌ Missing physical | High |
| Loss Given Default | `loss_given_default`, `loss_given_default_detail` | 🟡 Physical only | No logical model entity |

### 2.3 Identity Strategy

- **SLV-001 violation**: No `<entity>_sk` column is present on any physical table. The MD5 surrogate key pattern is entirely absent.
- **SLV-002 violation**: No `<entity>_bk` natural key column is defined on any physical table.
- The `loan_account_master` physical columns use business identifiers such as `CIF_NO`, `Account number`, `AGREEMENTID` as implicit natural keys, but none are formally designated or named per convention.

### 2.4 SCD Strategy

- **SLV-003 violation**: No `effective_from`, `effective_to`, or `is_current` columns exist on any physical table.
- `loan_account_master` table properties include `delta.enableChangeDataFeed=true` and `delta.enableDeletionVectors=true`, which suggests CDF is being used as a proxy for history tracking — but this is not equivalent to proper SCD-2 and downstream Gold consumers cannot query point-in-time states.
- `loan_term_account` and `loan_account` have only `deletionVectors` feature, no CDF — no history possible.

### 2.5 Cross-Entity Referential Integrity

| Relationship | Expected FK | Status |
|---|---|---|
| `loan_account` → `party` (customer) | `party_sk` or `customer_bk` | ❌ No FK; raw `Customer number` / `CIF_NO` embedded |
| `loan_account` → `product` | `product_sk` | ❌ No FK; raw `Product Code` embedded |
| `loan_account` → `account` (SA boundary) | `account_sk` | ❌ Not present |
| `loan_transaction` → `loan_account` | `loan_account_sk` | ❌ No surrogate FK |
| `loan_term_account` → `loan_account` | `loan_account_sk` | ❌ No surrogate FK |
| `loan_provision_*` → `loan_account` | `loan_account_sk` | ❌ No physical tables, no FK |

### 2.6 ETL Mapping Completeness

| Entity | Total Attrs | Mapped | Pending/Not Available | Coverage |
|---|---|---|---|---|
| Loan_Account | 144 | 125 | 19 | 87% |
| Loan_Transaction | 43 | 37 | 6 | 86% |
| Loan_Term_Account | 14 | 14 | 0 | 100% |
| Credit_Agreement | 13 | 12 | 1 | 92% |
| Loan_Provision_Balance_History | 7 | 5 | 2 | 71% |
| Loan_Provision_Change_History | 8 | 8 | 0 | 100% |
| Loan_Instalment_Schedule | 18 | 0 | 18 | 0% — no mapping sheet |
| Loan_Statement | 31 | 0 | 31 | 0% — no mapping sheet |
| Loan_Statement_Line_Item | 21 | 0 | 21 | 0% — no mapping sheet |

---

## 3. Entity Inventory

| # | Logical Entity | Physical Table | Table Status | Phys. Active? | Mapping Coverage | Assessment |
|---|---|---|---|---|---|---|
| 1 | Loan_Account | `loan_account_master` | 🟡 Wrong name | ✅ Active (204 MB) | 87% | Critical issues |
| 1b | — | `loan_account` | 🔴 Abandoned | ❌ Empty (0 B) | — | Deprecated shell |
| 2 | Loan_Transaction | `loan_transaction` | 🟡 Acceptable | ✅ Active (232 KB) | 86% | Issues |
| 3 | Loan_Term_Account | `loan_term_account` | ✅ Correct | ✅ Active (47 MB) | 100% | Issues |
| 4 | Credit_Agreement / Loan_Agreement | None | 🔴 Missing | ❌ — | 92% | Critical |
| 5 | Loan_Instalment_Schedule | None | 🔴 Missing | ❌ — | 0% | Critical |
| 6 | Loan_Statement | None | 🔴 Missing | ❌ — | 0% | Critical |
| 7 | Loan_Statement_Line_Item | None | 🔴 Missing | ❌ — | 0% | Critical |
| 8 | Loan_Provision_Balance_History | None | 🔴 Missing | ❌ — | 71% | High |
| 9 | Loan_Provision_Change_History | None | 🔴 Missing | ❌ — | 100% | High |
| 10 | Loan_GL_Account | None | 🔴 Missing | ❌ — | Partial | Medium |
| 11 | Provision_for_Loan_Loss_Account | None | 🔴 Missing | ❌ — | Partial | Medium |
| 12 | Allowance_for_Loan_Loss_Account | None | 🔴 Missing | ❌ — | Partial | Medium |
| 13 | Loan_Transaction_Account | None | 🔴 Missing | ❌ — | 100% | High |
| 14 | 20× Reference_* entities | `reference_branch_code`, `reference_currency_code`, `reference_product_loan` (3 of 20) | 🔴 Wrong schema | Partial | Partial | High — wrong schema |
| 15 | *(No logical model)* | `loss_given_default` | 🟡 Orphan | ✅ Active (11 MB) | — | Governance gap |
| 16 | *(No logical model)* | `loss_given_default_detail` | 🟡 Orphan | ✅ Active (15 MB) | — | Governance gap |
| 17 | *(No logical model)* | `account_loss_given_default_detail` | 🟡 Orphan | ✅ Active (735 KB) | — | Governance gap |
| 18 | *(No logical model)* | `pbd_c3_loan` | 🔴 Non-standard | ❌ Empty | — | Governance gap |
| 19 | *(No logical model)* | `pbd_hio_loan_account` | 🔴 Non-standard | ❌ Empty | — | Governance gap |
| 20 | IFRS 9 ECL Staging | None | 🔴 Entirely absent | ❌ — | 0% | **Critical regulatory** |

---

## 4. Per-Entity Assessments

---

### 4.1 Entity: Loan_Account → Physical: `loan_account_master`

#### 4.1a Industry Fitness

**Finding**: `Loan_Account` is a 141-attribute "god table" that conflates at least 8 distinct banking concerns in a single entity, violating 3NF:

1. **Core loan facility** (account number, customer, product, status, open/close dates)
2. **Balance sheet amounts** (principal balance, outstanding charges, accrued interest)
3. **Disbursement history** (first/last/full disbursement, MTD/YTD/LTD totals)
4. **Repayment terms** (instalment amounts, payment dates, due days)
5. **Interest rate structure** (effective rate, spread, index, next-tier rate)
6. **Collection status** (DPD, past due amounts, collection status code, bucket counts)
7. **Credit scoring** (account score, model ID, model run date)
8. **Product-specific tuition loan attributes** (course duration, type, faculty, graduation status)
9. **Product-specific car loan attributes** (car status, car status start date)
10. **ESG attributes** (ESG interest ratio, ESG COF, ESG status)

In BIAN terminology, this merges the **Loan** service domain with **Consumer Loan**, **Credit Risk Assessment**, **Collections**, and **ESG Portfolio** concepts — all in one table.

**Grain confusion**: The entity appears to be one row per loan account, but the presence of MTD/YTD/LTD aggregates implies it is being used as a point-in-time snapshot (period-grain), which contradicts an account-master grain.

#### 4.1b Attribute-Level Review

| Attribute | Logical Type | Mapping Status | Physical Concern |
|---|---|---|---|
| Account number | STRING | Mapped (RLS: `LEA_AGREEMENT_DTL.AGREEMENTID`, Flex: `VW_LOAN_ACCOUNT.APPLICATION_NUM`) | 🔴 **Ambiguous**: agreement ID ≠ application number — two different concepts mapped to same attribute |
| Account type | STRING | Mapped | 🟡 Should be `loan_account_type_code` (NAM-001) |
| Account open date | STRING | Mapped | 🔴 Type should be `DATE` not `STRING` |
| Account close date | STRING | Mapped | 🔴 Type should be `DATE` not `STRING` |
| Principal balance | STRING | Mapped | 🔴 Type should be `DECIMAL(18,4)` per DQ Gate 2; naming should be `principal_balance_amount_aed` |
| ALL monetary amounts (25+) | STRING | Various | 🔴 All must be `DECIMAL(18,4)` |
| ALL date attributes (30+) | STRING | Various | 🔴 All must be `DATE` or `TIMESTAMP` |
| Days past due count | STRING | Mapped | 🔴 Type should be `INT` |
| Number of times 10–29/30–59/60–89/90+ DPD | STRING | Mapped | 🔴 Type should be `INT`; **aggregated counts violate SLV-007** |
| LEFS mtd disbursement amount | STRING | Mapped | 🔴 MTD/YTD/LTD = pre-aggregated metrics, must NOT be in Silver (SLV-007) |
| LEFS ytd disbursement amount | STRING | Mapped | 🔴 Same — SLV-007 violation |
| LEFS Itd disbursement amount | STRING | Mapped | 🔴 Same — SLV-007 violation |
| MTD/YTD/LTD disbursement paid prn amount | STRING | Mapped | 🔴 Same — SLV-007 violation |
| MTD/YTD/LTD disbursement paid Ic amount | STRING | Mapped | 🔴 Same — SLV-007 violation |
| ESG interest ratio | STRING | Not Available | 🔴 Unmapped; ESG concern should be in separate entity |
| ESG fir rt | STRING | Not Available | 🔴 Unmapped; abbreviated name violates NAM-001 |
| ESG cof | STRING | Pending | 🔴 Unmapped; abbreviated name violates NAM-001 |
| ESG status code | STRING | Not Available | 🔴 Unmapped |
| Tuition status code | STRING | Pending | 🟡 Product-specific attribute mixed in account master |
| Car status code | STRING | Mapped | 🟡 Product-specific attribute mixed in account master |
| Account Score Value | STRING | Mapped | 🟡 Risk scoring concern should be in separate entity |
| Model id / run id | STRING | Mapped | 🟡 Scoring model metadata in account table |
| Source System Code | STRING | Mapped (no source column) | 🔴 Mapped but no actual source value — metadata column treatment |
| Business Start Date | CHAR(18) | — | 🔴 Non-standard name; should be `effective_from` (TIMESTAMP) |
| Business End Date | CHAR(18) | — | 🔴 Non-standard name; should be `effective_to` (TIMESTAMP) |
| Is Active Flag | CHAR(18) | — | 🟡 Type should be `STRING` not `CHAR(18)`; should be `is_active_flag` |
| IsLogicallyDeleted | STRING | — | 🟡 Non-standard name; should be `delete_date` (TIMESTAMP) with actual timestamp |

#### 4.1c Metadata Completeness

**All 11 mandatory Silver audit columns are absent** from `loan_account_master` per the physical mapping analysis:

| Required Column | Standard Type | Status |
|---|---|---|
| `source_system_code` | STRING | ❌ Missing |
| `source_system_id` | STRING | ❌ Missing |
| `create_date` | TIMESTAMP | ❌ Missing |
| `update_date` | TIMESTAMP | ❌ Missing |
| `delete_date` | TIMESTAMP | ❌ Missing |
| `is_active_flag` | STRING | ❌ Missing (logical model has `Is Active Flag` CHAR(18) — non-standard) |
| `effective_from` | TIMESTAMP | ❌ Missing (logical model has `Business Start Date` CHAR(18) — wrong type and name) |
| `effective_to` | TIMESTAMP | ❌ Missing (logical model has `Business End Date` CHAR(18) — wrong type and name) |
| `is_current` | BOOLEAN | ❌ Missing |
| `run_id` | STRING | ❌ Missing |
| `source_ingestion_date` | TIMESTAMP | ❌ Missing |

---

### Finding LoanAccount-001 — God Table: Loan_Account Mixes 8+ Distinct Concerns

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | `SLV-006` — "3NF; no derived values/aggregations"; Guideline §2.1 — "Decompose entities into 3NF" |
| Evidence | Logical model: `Loan_Account` 141 attrs · Physical: `loan_account_master` 212 physical cols |
| Affected Table | `silver.loan.loan_account_master` |
| Affected Column(s) | All tuition/car/ESG/credit-scoring/collection/disbursement/MTD-YTD-LTD columns |
| Confidence | 0.97 |

**Description:** `Loan_Account` contains 141 logical attributes spanning core facility data, repayment terms, collection status, credit scoring, product-specific tuition loan fields, car loan fields, ESG indicators, and period-aggregated disbursement metrics. This violates Third Normal Form, mixes transactional with reference concerns, and makes the table a "god table" anti-pattern. The physical implementation `loan_account_master` has 212 columns, confirming the problem. Downstream impact: analytics requiring individual attribute history (e.g., "what was the collection status on 2025-03-01?") are impossible without SCD-2 on a properly scoped entity.

**Remediation:** Decompose into: (a) `loan_account` — core facility, (b) `loan_collection_status` — DPD, collection codes, bucket counts, (c) `loan_credit_score` — scoring model data, (d) `loan_disbursement` — atomic disbursement events, (e) `loan_esg_indicator` — ESG classification. Remove all MTD/YTD/LTD aggregates (Gold layer only).

**Estimated Effort:** Large
**Owner:** Data Modeling

---

### Finding LoanAccount-002 — All Monetary Amounts Typed as STRING

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | DQ Gate 2 — "All monetary columns must use `DECIMAL(18,4)`" |
| Evidence | Logical model: all 25+ monetary attributes in Loan_Account have type `STRING` · Mapping: source types are NUMBER/DECIMAL |
| Affected Table | `silver.loan.loan_account_master` |
| Affected Column(s) | `Principal balance`, `Account currency original commitment`, `Outstanding principal AED amount`, `Installment amount`, all ~25 monetary columns |
| Confidence | 1.0 |

**Description:** Every monetary attribute in the Loan_Account entity (and across all loan entities) is typed `STRING` in the logical model. Source systems emit `NUMBER`, `DECIMAL`, and `NUMBER(x,y)` types. Storing monetary amounts as STRING prevents arithmetic operations, causes implicit cast failures at Gold, masks truncation, and violates the data quality gate requirement that monetary columns use `DECIMAL(18,4)`.

**Remediation:** Re-type all monetary attributes to `DECIMAL(18,4)`. Currency-specific columns should follow the `<measure>_amount_aed` or `<measure>_amount_orig` naming pattern. Rename columns accordingly per NAM-001.

**Estimated Effort:** Large
**Owner:** Data Modeling

---

### Finding LoanAccount-003 — All Date/Time Attributes Typed as STRING

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | DQ Gate 2 — "All date columns must conform to `YYYY-MM-DD` format"; Guideline §2.9 — "All timestamps stored in UTC" |
| Evidence | Logical model: all 30+ date/time attributes have type `STRING` |
| Affected Table | `silver.loan.loan_account_master`, `silver.loan.loan_transaction`, `silver.loan.loan_term_account` |
| Affected Column(s) | `Account open date`, `Account close date`, `Loan date`, `First disbursement date`, `Loan settlement date`, all ~30 date columns |
| Confidence | 1.0 |

**Description:** All 30+ date and datetime attributes across loan entities are typed `STRING` in the logical model. This prevents date arithmetic, range filters, and temporal queries. Source system types include `DATE` and `TIMESTAMP` types. This is a systemic modeling deficiency.

**Remediation:** Re-type: business dates → `DATE` with suffix `_date`, timestamps → `TIMESTAMP` with suffix `_timestamp` and UTC normalization. Apply naming convention (`account_open_date`, `loan_disbursement_date`).

**Estimated Effort:** Large
**Owner:** Data Modeling

---

### Finding LoanAccount-004 — MTD/YTD/LTD Aggregates Must Not Be in Silver

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | `SLV-007` — "No pre-computed metrics"; Guideline §2.6 — "Store only atomic, un-aggregated facts in Silver" |
| Evidence | Logical model: `Loan_Account` includes `LEFS mtd disbursement amount`, `LEFS ytd disbursement amount`, `LEFS Itd disbursement amount`, `MTD disbursement paid prn amount`, `YTD disbursement paid prn amount`, `LTD disbursement paid prn amount`, `MTD disbursement paid Ic amount`, `YTD disbursement paid Ic amount`, `LTD disbursement paid Ic amount` |
| Affected Table | `silver.loan.loan_account_master` |
| Affected Column(s) | 9 MTD/YTD/LTD columns as listed above |
| Confidence | 1.0 |

**Description:** Nine period-aggregate metrics (month-to-date, year-to-date, life-to-date) are embedded in the Silver `Loan_Account` entity. These are derived by aggregating over underlying atomic disbursement events. They must not exist in Silver; they belong in the Gold layer.

**Remediation:** Remove all 9 MTD/YTD/LTD columns from Silver. Create a `loan_disbursement` entity in Silver with one row per disbursement event. Build Gold aggregates from that.

**Estimated Effort:** Medium
**Owner:** Data Modeling + ETL

---

### Finding LoanAccount-005 — No Surrogate Key or Business Key

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | `SLV-001` — "`<entity>_sk` (MD5) on every entity"; `SLV-002` — "`<entity>_bk` on every entity" |
| Evidence | Physical mapping (`loan_account_master` 212 columns): no `loan_account_sk`, no `loan_account_bk` column present |
| Affected Table | `silver.loan.loan_account_master` |
| Affected Column(s) | Missing `loan_account_sk`, `loan_account_bk` |
| Confidence | 1.0 |

**Description:** Neither a deterministic MD5 surrogate key (`loan_account_sk`) nor a canonical business key (`loan_account_bk`) is defined on the physical table. The table relies on raw source-system identifiers such as `CIF_NO` and `Account number`, which differ between RLS and Flexcube sources (AGREEMENTID vs. APPLICATION_NUM). Without `loan_account_sk`, FK relationships to this entity from Gold layer facts are impossible.

**Remediation:** Define `loan_account_bk` as the canonical loan account number (resolve the RLS vs. Flexcube ambiguity). Add `loan_account_sk = MD5(UPPER(TRIM(loan_account_bk)))` as first column (NOT NULL, PRIMARY KEY).

**Estimated Effort:** Large
**Owner:** Data Modeling + ETL

---

### Finding LoanAccount-006 — Complete Absence of Mandatory Audit Columns

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | Guideline §2.7 — "Include the following Silver technical audit columns on every Silver entity table"; `SLV-008` — "All metadata columns present" |
| Evidence | Physical mapping: none of the 11 required audit columns found; logical model uses non-standard `Business Start Date` (CHAR(18)), `Business End Date` (CHAR(18)), `IsLogicallyDeleted` (STRING) |
| Affected Table | `silver.loan.loan_account_master` |
| Affected Column(s) | All 11: `source_system_code`, `source_system_id`, `create_date`, `update_date`, `delete_date`, `is_active_flag`, `effective_from`, `effective_to`, `is_current`, `run_id`, `source_ingestion_date` |
| Confidence | 0.95 |

**Description:** The physical table `loan_account_master` has none of the 11 mandatory Silver audit columns. The logical model partially addresses this with `Business Start Date` and `Business End Date` (which are non-standard names for `effective_from`/`effective_to`) and uses `CHAR(18)` type (incorrect — should be TIMESTAMP). `IsLogicallyDeleted` is used instead of the correct `delete_date` (TIMESTAMP). Without these columns there is no SCD-2 history, no pipeline lineage, no DQ status tracking, and no soft-delete capability.

**Remediation:** Add all 11 columns with correct names and types. Rename `Business Start Date` → `effective_from TIMESTAMP`, `Business End Date` → `effective_to TIMESTAMP`, `IsLogicallyDeleted` → `delete_date TIMESTAMP`. Add missing: `source_system_code`, `source_system_id`, `create_date`, `update_date`, `is_active_flag`, `is_current`, `run_id`, `source_ingestion_date`.

**Estimated Effort:** Medium
**Owner:** Data Modeling + DBA

---

### Finding LoanAccount-007 — Ambiguous Account Number Mapping (RLS vs. Flexcube)

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | High |
| Guideline Rule | Guideline §2.10 — "Create a canonical base entity plus per-source companion tables" |
| Evidence | Mapping: `Account number` → RLS: `LEA_AGREEMENT_DTL.AGREEMENTID` (agreement-level natural key) vs. Flexcube: `VW_LOAN_ACCOUNT.APPLICATION_NUM` (application number) |
| Affected Table | `silver.loan.loan_account_master` |
| Affected Column(s) | `Account number` |
| Confidence | 0.93 |

**Description:** The `Account number` attribute maps to `AGREEMENTID` in RLS (the agreement-level identifier) and `APPLICATION_NUM` in Flexcube (the application-level identifier). These are semantically different keys representing different lifecycle stages of a credit facility. Conflating them into a single `Account number` column will produce incorrect joins, deduplication failures, and downstream key resolution errors.

**Remediation:** Resolve which identifier is the canonical loan account BK. Candidates: (a) use `AGREEMENTID` as the BK (agreement is the post-approval binding contract), (b) create separate BK columns `rls_account_bk` and `flex_account_bk` with a canonical `loan_account_bk` resolved via MDM logic. Document the resolution clearly in the mapping.

**Estimated Effort:** Medium
**Owner:** Data Modeling + ETL + Governance

---

### Finding LoanAccount-008 — Table Name Non-Compliance: loan_account_master

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | Medium |
| Guideline Rule | `NAM-003` — "Silver entity naming: `<entity>`"; table pattern for Silver is `<entity>`, not `<entity>_master` |
| Evidence | Physical: table is `silver_dev_v2.loan.loan_account_master`; standard requires `silver.loan.loan_account` |
| Affected Table | `silver.loan.loan_account_master` |
| Affected Column(s) | Table name |
| Confidence | 1.0 |

**Description:** The active loan account table is named `loan_account_master` which violates the Silver table naming convention (`<entity>` = `loan_account`). A separate empty `loan_account` table exists in the catalog (created Dec 2025, 0 bytes, 0 files) suggesting the correct name was once planned. Having both `loan_account` (empty) and `loan_account_master` (active) creates catalog confusion, incorrect lineage, and will cause consumers to query the wrong table.

**Remediation:** Rename `loan_account_master` → `loan_account`. Deprecate and drop the empty `loan_account` shell. Follow the breaking-change migration procedure (create `loan_account_v2`, migrate data, update pipeline references, deprecate old table for 6 months).

**Estimated Effort:** Medium
**Owner:** DBA + ETL

---

### Finding LoanAccount-009 — Column Naming: PascalCase / Mixed Case Violations

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | Medium |
| Guideline Rule | `NAM-001` — "Use snake_case for all identifiers; use lowercase only" |
| Evidence | Physical mapping: `Accred_LPI`, `AUTHORIZED_ON`, `CIF_NO`, `DPD_STRING`, `NPA_Stage_Identifier`, `NFP_Flag`, `DSA_CSA`, `LAA_SOURCE_CHANNEL_PARTNER_C`, `LAD_DOCSTAGE_C` — all violate lowercase snake_case |
| Affected Table | `silver.loan.loan_account_master` |
| Affected Column(s) | ~40 columns with PascalCase, SCREAMING_CASE, or mixed identifiers |
| Confidence | 1.0 |

**Description:** The `loan_account_master` physical table contains numerous column names that directly copy source-system naming conventions (Oracle uppercase column names, proprietary Flexcube/RLS codes). This violates the platform's snake_case lowercase convention and makes SQL queries case-sensitive in some engines. `CIF_NO` should be `cif_no` or better `customer_bk`; `LPI` and `DPD_STRING` should be spelled out (`late_payment_indicator`, `days_past_due_string`).

**Remediation:** Create a column rename migration. Use approved abbreviations from the naming convention table. Replace `SCREAMING_CASE` with `snake_case`. Replace abbreviations: `DPD` → `days_past_due`, `EMI` → `instalment_amount_aed` (or `emi_amount_aed`), `CIF_NO` → `customer_bk`, `NPA` → retain as approved abbreviation if in the approved list (not listed — spell out as `non_performing_asset`).

**Estimated Effort:** Medium
**Owner:** Data Modeling

---

### 4.2 Entity: Loan_Transaction → Physical: `loan_transaction`

#### 4.2a Industry Fitness

`Loan_Transaction` correctly represents atomic posting events against a loan account. However, there is a **Subject Area boundary issue**: the Transaction SA (`silver.transaction`) is intended as the canonical home for all double-entry financial postings. Having `loan_transaction` as a separate Silver table creates duplication risk — loan transactions will also appear in `silver.transaction`. Confirm with the data steward whether `silver.loan.loan_transaction` is a staging/filtered view or intended to be the authoritative record.

#### 4.2b Attribute-Level Review

| Attribute | Logical Type | Mapping Status | Physical Concern |
|---|---|---|---|
| Transaction Id | STRING | Mapped | 🔴 No `loan_transaction_sk`, no `loan_transaction_bk` defined |
| Transaction Date | STRING | Not Available (RLS) / Mapped (Flex) | 🔴 Type must be `DATE`; `Transaction Time` should be merged into `transaction_timestamp TIMESTAMP` |
| Transaction Amount | STRING | Mapped | 🔴 Must be `DECIMAL(18,4)`, name should be `transaction_amount_orig` |
| Base Currency Transaction Amount | STRING | Mapped | 🔴 Must be `DECIMAL(18,4)`, name should be `transaction_amount_aed` |
| Segment / Segment Description | STRING | Mapped | 🟡 Denormalization — customer segment should not be in transaction; violates 3NF |
| Transaction Your Reference | STRING | Pending | 🟡 Ambiguous — pending for modelers |
| Transaction Our Reference | STRING | Pending | 🟡 Same source as `Transaction Your Reference` — unclear distinction |
| Sub Sequence Id | STRING | Pending/Not Available | 🟡 Pending — needs business clarification |
| Source System Code | STRING | Pending | 🔴 SLV-008 metadata column must be mandatory and pre-populated by pipeline |
| Source System Id | STRING | Pending | 🔴 Same |
| IsLogicallyDeleted | STRING | Mapped (no source) | 🔴 Non-standard column name; should be `delete_date TIMESTAMP` |
| Business Start Date | CHAR(18) | — | 🔴 Wrong name and type; must be `effective_from TIMESTAMP` |
| Business End Date | CHAR(18) | — | 🔴 Wrong name and type; must be `effective_to TIMESTAMP` |

#### 4.2c Metadata Completeness

Same systemic gap as Loan_Account — `Business Start Date`/`Business End Date` (CHAR(18)) and `IsLogicallyDeleted` are non-standard proxies that do not meet the §2.7 mandatory audit column requirement.

---

### Finding LoanTransaction-001 — Transaction SA Boundary Overlap

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | Medium |
| Guideline Rule | Guideline §2.2 — "Keep each Silver subject area self-contained" |
| Evidence | `sa/subject-areas.md`: SA #12 Transaction = "Atomic Double-Entry Postings"; `silver.loan.loan_transaction` duplicates this scope |
| Affected Table | `silver.loan.loan_transaction` |
| Affected Column(s) | All |
| Confidence | 0.85 |

**Description:** `silver.loan.loan_transaction` overlaps with `silver.transaction.financial_transaction`. Both claim loan repayment/posting events. Without a clear boundary agreement, loan postings may be loaded into both tables. If `loan_transaction` is intentional (loan-specific enriched view), it must be documented as a filtered projection with loan-specific attributes only; the canonical atomic postings should remain in the Transaction SA.

**Remediation:** Define in governance whether `loan_transaction` is (a) a loan-filtered projection of `silver.transaction` (in which case it is a Gold-layer view concern, not a Silver entity), or (b) a loan-specific enriched entity with additional loan attributes. Document the decision; if (a), deprecate the Silver entity.

**Estimated Effort:** Small
**Owner:** Governance + Data Modeling

---

### Finding LoanTransaction-002 — Segment Denormalization in Transaction Entity

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | Medium |
| Guideline Rule | `SLV-006` — "3NF; no derived values"; Guideline §2.1 |
| Evidence | Logical model: `Loan_Transaction` contains `Segment` and `Segment Description` columns |
| Affected Table | `silver.loan.loan_transaction` |
| Affected Column(s) | `Segment`, `Segment Description` |
| Confidence | 1.0 |

**Description:** Customer segment data is embedded in the transaction entity. Segment belongs to the Party/Customer entity and is a slowly-changing attribute. Including it in a transaction record (which should be immutable) creates a denormalization that will produce incorrect historical analyses (segment changes will not retroactively update transaction records).

**Remediation:** Remove `Segment` and `Segment Description` from `loan_transaction`. Joins to segment are performed at Gold layer via `party_sk` → `dim_customer`.

**Estimated Effort:** Small
**Owner:** Data Modeling

---

### 4.3 Entity: Loan_Term_Account → Physical: `loan_term_account`

#### 4.3a Industry Fitness

`Loan_Term_Account` correctly captures loan-contract-level term attributes (maturity date, amortization method, balloon amounts, original loan amount, renewal dates). Grain is one row per loan contract term period. Entity boundary is reasonable.

**Concern**: `loan_term_account` has `6,654 files` (47 MB), suggesting many small files due to per-row or micro-batch writes — performance issue but not a modeling gap.

#### 4.3b Attribute-Level Review

| Attribute | Logical Type | Mapping Status | Physical Concern |
|---|---|---|---|
| Account Number | STRING | Mapped | 🔴 No `loan_term_account_sk`, no FK `loan_account_sk` |
| Original Loan Amount | STRING | Mapped (both sources) | 🔴 Must be `DECIMAL(18,4)`, rename to `original_loan_amount_aed` |
| Balloon Amount | STRING | Mapped | 🔴 Must be `DECIMAL(18,4)`, rename to `balloon_amount_aed` |
| Amortization Method Code | STRING | Mapped | 🟡 Name is correct; should be `amortization_method_code STRING` |
| Loan Maturity Date | STRING | Mapped | 🔴 Must be `DATE` → `loan_maturity_date` |
| Loan Renewal Date | STRING | Mapped (ambiguous: RLS maps to `RESCHEDULEDATE`, Flex maps to `MIGRATION_DATE`) | 🟡 Semantically ambiguous — reschedule ≠ migration |
| Commit Start Date / Commit End Date | STRING | Mapped | 🔴 Must be `DATE` |
| Account Cmcy Balloon Amount | STRING | Mapped | 🟡 Duplicate of `Balloon Amount` in account currency — verify if truly different (e.g., LCY vs. FCY) |
| Account Cmcy Original Loan Amount | STRING | Mapped (Flex: `VW_LOAN_ACCOUNT.TOTAL_FINANCE_OUTSTANDING_AED`) | 🔴 Outstanding ≠ original — mapping is semantically incorrect for Flexcube |

#### 4.3c Metadata Completeness

`loan_term_account` physical table has only `deletionVectors` feature enabled — no `changeDataFeed`, no `rowTracking`. Standard audit columns and SCD-2 columns must be added. No metadata columns are visible in the mapping.

---

### Finding LoanTermAccount-001 — Flexcube Mapping Semantic Error: Outstanding ≠ Original

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | High |
| Guideline Rule | Behavioral rule — "Do not assume correct mapping based on similar column names — verify type, nullability, and semantics" |
| Evidence | Mapping: `Account Cmcy Original Loan Amount` → Flexcube: `VW_LOAN_ACCOUNT.TOTAL_FINANCE_OUTSTANDING_AED` |
| Affected Table | `silver.loan.loan_term_account` |
| Affected Column(s) | `Account Cmcy Original Loan Amount` |
| Confidence | 0.92 |

**Description:** The Flexcube source column `TOTAL_FINANCE_OUTSTANDING_AED` represents the **current outstanding balance**, not the **original sanctioned amount**. This is a critical semantic mapping error — the original amount is fixed at origination, while outstanding changes with each repayment. Using outstanding as "original" will produce incorrect LTV, utilization, and portfolio analytics.

**Remediation:** Identify the correct Flexcube source for original loan amount (likely `CLTB_ACCOUNT_APPS_MASTER.AMOUNT_DISBURSED` or `N_SANCTIONED_AMOUNT` from FCT_LOANS_CONTRACTS). Correct the mapping before pipeline deployment.

**Estimated Effort:** Small
**Owner:** ETL + Data Modeling

---

### 4.4 Entity: Credit_Agreement / Loan_Agreement — **Physical Table Missing**

#### 4.4a Industry Fitness

`Loan_Agreement` (also named `Credit_Agreement` in the data mapping) represents the **legal contract** between bank and borrower — a critical first-class entity in any banking data model. It carries seniority level, re-aging count, past-due amounts, charge-off amounts, category code, and borrower purpose. This is distinct from the operational `Loan_Account` (which tracks balances and status) and from `Loan_Term_Account` (which tracks repayment schedule parameters).

#### 4.4b Physical Status

**No physical table exists** for `loan_agreement` / `credit_agreement` in `silver.loan`. The data mapping has 13 attributes fully or partially mapped (92% coverage), but the table has never been created.

---

### Finding LoanAgreement-001 — Critical Entity Missing Physical Table

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | `SLV-001` through `SLV-008` — all apply once table is created; `SLV-003` — entity must have SCD-2 |
| Evidence | Physical structures CSV: no `loan_agreement` or `credit_agreement` table · Mapping sheet: `Credit_Agreement` 13 attrs with 92% mapping |
| Affected Table | `silver.loan.loan_agreement` (to be created) |
| Affected Column(s) | All 13 mapped attributes |
| Confidence | 1.0 |

**Description:** The legal agreement entity has no physical DDL in production. The mapped attributes include charge-off amounts, reaging count, seniority level, and obligor purpose — all critical for BCBS 239 risk data aggregation and IFRS 9 staging. The entity name diverges between logical model (`Loan_Agreement`) and data mapping (`Credit_Agreement`); the canonical Silver name must be `loan_agreement`.

**Remediation:** Create `silver.loan.loan_agreement` with: `loan_agreement_sk`, `loan_account_bk`, all 13 mapped attributes (with corrected types), plus 11 audit columns. Note: `Seniority Level Code` is Not Available in RLS — ensure Flexcube source (`CSTB_UPLOAD_LIQ_ADVICES.PRIORITY`) is confirmed.

**Estimated Effort:** Medium
**Owner:** DBA + ETL

---

### 4.5 Entity: Loan_Instalment_Schedule — **Physical Table Missing, No Mapping**

#### 4.5a Industry Fitness

`Loan_Instalment_Schedule` stores the contractual repayment schedule — one row per instalment per loan. This is essential for: (a) cash flow projection, (b) IFRS 9 expected cash flow modelling, (c) overdue detection (compare actual vs. scheduled). It is a **critical** entity for a lending data model.

#### 4.5b Physical Status

**No physical table** and **no data mapping** exists for this entity. The logical model defines 18 attributes.

---

### Finding LoanInstalmentSchedule-001 — Critical Entity: No Physical Table, No Mapping

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | General principle — "Missing entity"; SLV-001 through SLV-008 |
| Evidence | Physical structures: no `loan_instalment_schedule` table · Data mapping: no entry for this entity |
| Affected Table | `silver.loan.loan_instalment_schedule` (to be created) |
| Affected Column(s) | All 18 attributes unmapped |
| Confidence | 1.0 |

**Description:** The repayment schedule is unmapped and has no physical implementation. Without it, it is impossible to determine scheduled vs. actual payment timings, days-past-due from first principles, or generate accurate cash flow projections required under IFRS 9. This also limits BCBS 239 credit risk data quality. Source likely: RLS `FIN_LEAM.LEA_REPAYSCH_DTL` (referenced in Loan_Account mapping for instalment-related attributes).

**Remediation:** Add mapping for all 18 attributes using `FIN_LEAM.LEA_REPAYSCH_DTL` and Flexcube equivalent. Create physical DDL with PK `loan_instalment_sk`, FK `loan_account_sk`, `instalment_number INT`, `due_date DATE`, `principal_amount_aed DECIMAL(18,4)`, `interest_amount_aed DECIMAL(18,4)`, `instalment_status_code STRING`.

**Estimated Effort:** Large
**Owner:** Data Modeling + ETL

---

### 4.6 Entity: Loan_Statement — **Physical Table Missing, No Mapping**

#### 4.6a Industry Fitness

`Loan_Statement` (31 attributes) represents periodic customer-facing statements. It enables customer-facing analytics and regulatory disclosure. Absence from the physical model means no statement history is persisted in Silver.

---

### Finding LoanStatement-001 — Entity Missing Physical Table and Mapping

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | Medium |
| Guideline Rule | Guideline §2.11 — "Tables that exist in platform but have no corresponding entity in Erwin are a governance gap" |
| Evidence | Physical structures: no `loan_statement` table · Data mapping: no entry |
| Affected Table | `silver.loan.loan_statement` (to be created) |
| Affected Column(s) | All 31 attributes unmapped |
| Confidence | 1.0 |

**Description:** No physical table or mapping exists for the loan statement entity. 31 logical attributes are defined. Customer statement data is not persisted in Silver, limiting customer service analytics and regulatory disclosure capabilities.

**Remediation:** Create mapping and physical DDL for `loan_statement` and `loan_statement_line_item`. Source candidates: CBS statement generation tables. Prioritize after P0 items.

**Estimated Effort:** Large
**Owner:** Data Modeling + ETL

---

### 4.7 Entity: Loan_Provision_Balance_History / Loan_Provision_Change_History — **Physical Tables Missing**

#### 4.7a Industry Fitness

Provision entities track ECL/NPA provisions — critical for IFRS 9 compliance and CBUAE regulatory reporting. `Loan_Provision_Balance_History` (period-end balance) and `Loan_Provision_Change_History` (change movements) together constitute the provision ledger. These are distinct from the `loss_given_default*` tables (which appear to cover LGD model outputs — different concept).

#### 4.7b Physical Status

No dedicated physical tables. Three LGD tables exist in the physical catalog (`loss_given_default`, `loss_given_default_detail`, `account_loss_given_default_detail`) but are **not described in the logical model** and are likely model outputs, not provision ledger entries.

---

### Finding LoanProvision-001 — Provision Entities Missing Physical Implementation

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | IFRS 9 / BCBS 239 principles; `SLV-001` through `SLV-008` |
| Evidence | Physical structures: no `loan_provision_balance_history` or `loan_provision_change_history` · Mapping: both have 71%–100% mapping coverage |
| Affected Table | `silver.loan.loan_provision_balance_history`, `silver.loan.loan_provision_change_history` (to be created) |
| Affected Column(s) | All |
| Confidence | 1.0 |

**Description:** ECL provision data is partially mapped (71%–100% coverage) but has no physical Silver table. This is a regulatory risk: IFRS 9 requires point-in-time provision balances with full audit trail. The `Geographical Area ID` and `Risk Scenario Id` attributes are Not Available from both sources — these may need to be sourced from the Risk SA or derived from reference data.

**Remediation:** Create DDL for both entities. For `Geographical Area ID` and `Risk Scenario ID` (unmapped), confirm with Risk SA steward whether these are FK references to `silver.risk` entities. Implement as nullable FKs pending resolution.

**Estimated Effort:** Large
**Owner:** Data Modeling + ETL + Risk team

---

### 4.8 Entity: Loss Given Default Tables — **Orphan Physical Tables (No Logical Model)**

Three physical tables exist with no corresponding logical model entity:
- `loss_given_default` (11 MB, active)
- `loss_given_default_detail` (15 MB, active)
- `account_loss_given_default_detail` (735 KB, active)

---

### Finding LGD-001 — Orphan Physical Tables Without Logical Model Entities

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | Medium |
| Guideline Rule | Guideline §2.11 — "Tables that exist in the platform but have no corresponding entity in the Erwin model are a governance gap"; Guideline §2.12 — catalog registration required |
| Evidence | Physical structures: three active LGD tables · Logical model: no matching entities |
| Affected Table | `silver.loan.loss_given_default`, `silver.loan.loss_given_default_detail`, `silver.loan.account_loss_given_default_detail` |
| Affected Column(s) | All |
| Confidence | 0.97 |

**Description:** Three LGD tables are deployed and active (with data) but are not represented in the logical model. This is a governance gap: the tables were built outside the approved modeling process. Without logical model entries, there is no stewardship, no data dictionary, no DQ rules, and no approved lineage. These tables also lack catalog descriptions (`description: null`).

**Remediation:** (a) Retroactively create logical model entities for LGD tables and add them to the Loan SA. (b) Add all mandatory audit columns (currently absent from the physical mapping). (c) Register in Informatica GDGC with steward, classification, and lineage.

**Estimated Effort:** Medium
**Owner:** Data Modeling + Governance

---

### 4.9 Entity: Reference_* (20 entities) — **Wrong Schema, Partial Physical**

#### 4.9a Industry Fitness

The Loan logical model contains 20 reference entities including `Reference_Currency`, `Reference_Bank_Branch`, `Reference_Loan_Status`, `Reference_Exchange_Rate`, etc. These are domain reference/lookup tables.

**Critical scope violation**: Per `sa/subject-areas.md`, reference/lookup data belongs in `silver.reference` (SA #1 — Reference Data). Three physical reference tables are co-located in `silver.loan`: `reference_branch_code`, `reference_currency_code`, `reference_product_loan`.

---

### Finding Reference-001 — Reference Tables in Wrong Subject Area Schema

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | High |
| Guideline Rule | `SLV-004` — "No source-system codes; use `silver.reference.code_mapping`"; SA taxonomy — reference data belongs in `silver.reference` |
| Evidence | Physical: `silver.loan.reference_branch_code`, `silver.loan.reference_currency_code`, `silver.loan.reference_product_loan` · Subject areas: `silver.reference` is the canonical home |
| Affected Table | All three `reference_*` tables in `silver.loan` |
| Affected Column(s) | N/A (table-level) |
| Confidence | 1.0 |

**Description:** Three reference tables are deployed in `silver.loan` instead of `silver.reference`. This violates subject-area isolation and creates duplicate reference data risk. Currency codes in `silver.loan.reference_currency_code` may diverge from the canonical set in `silver.reference`. The 17 remaining reference entities in the logical model have no physical tables at all.

**Remediation:** Move/migrate the three tables to `silver.reference`. For the remaining 17 reference entities, evaluate whether they should use the canonical `silver.reference.code_mapping` table (preferred) or have dedicated reference tables in `silver.reference`. Do not create reference tables in `silver.loan`.

**Estimated Effort:** Medium
**Owner:** Data Modeling + ETL + Reference Data team

---

### 4.10 Entity: Credit_Card_Statement / Credit_Card_Statement_Line_Item — **Wrong Subject Area**

---

### Finding CardStatement-001 — Card Entities in Loan SA

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | Medium |
| Guideline Rule | Guideline §2.2 — "Keep each Silver subject area self-contained" |
| Evidence | Logical model Loan SA: `Credit_Card_Statement` (31 attrs), `Credit_Card_Statement_Line_Item` (26 attrs) · `sa/subject-areas.md`: Card SA = `silver.card` |
| Affected Table | N/A (not yet physically created) |
| Affected Column(s) | All |
| Confidence | 1.0 |

**Description:** Two card statement entities are incorrectly modelled within the Loan subject area. These belong in the Card SA (`silver.card`). Their inclusion in the Loan SA inflates scope, creates cross-SA governance confusion, and will cause Card SA physical entities to be placed in `silver.loan` instead of `silver.card`.

**Remediation:** Move `Credit_Card_Statement` and `Credit_Card_Statement_Line_Item` to the Card SA logical model. Remove from Loan SA.

**Estimated Effort:** Small
**Owner:** Data Modeling + Governance

---

### 4.11 Missing Critical Entity: IFRS 9 ECL Staging

---

### Finding IFRS9-001 — ECL Staging Entity Entirely Absent

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | `sa/subject-areas.md` SA 8a — "IFRS 9 ECL Staging" explicitly listed as a required entity; BCBS 239 Principle 3 (accuracy), IFRS 9 — ECL measurement requires staging history |
| Evidence | Logical model: no `ifrs9_staging` or `ecl_staging` entity · Physical structures: no such table · Data mapping: no entry |
| Affected Table | `silver.loan.ifrs9_ecl_staging` (to be created) |
| Affected Column(s) | All |
| Confidence | 1.0 |

**Description:** The SA taxonomy explicitly requires IFRS 9 ECL Staging in the Loan SA. Neither the logical model nor the physical structures contain any ECL staging entity. ECL staging captures: IFRS 9 Stage (1/2/3), PD estimates, LGD estimates, EAD, ECL amount, staging date, staging trigger reason, and effective period. Without this, the bank cannot produce IFRS 9-compliant ECL disclosures directly from the Silver layer. Note: LGD tables do exist physically (addressed in 4.8) but do not substitute for a complete ECL staging entity which requires all three components (PD, LGD, EAD) plus staging classification.

**Remediation:** Design and implement `silver.loan.ifrs9_ecl_staging` with grain = one row per loan account per reporting date per scenario. Attributes: `ifrs9_staging_sk`, `loan_account_bk`, `reporting_date`, `ifrs9_stage_code` (1/2/3), `pd_estimate`, `lgd_estimate`, `ead_amount_aed`, `ecl_amount_aed`, `staging_trigger_code`, `model_id`, `model_run_date`, plus all 11 audit columns.

**Estimated Effort:** Large
**Owner:** Data Modeling + Risk team + ETL

---

### 4.12 Entities: pbd_c3_loan / pbd_hio_loan_account — **Empty Non-Standard Tables**

---

### Finding PBD-001 — Non-Standard PBD Staging Tables

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | Low |
| Guideline Rule | `NAM-003` — Silver entity naming pattern is `<entity>`; no `pbd_` prefix defined |
| Evidence | Physical: `silver.loan.pbd_c3_loan` (0 files, 0 bytes), `silver.loan.pbd_hio_loan_account` (0 files, 0 bytes) |
| Affected Table | `silver.loan.pbd_c3_loan`, `silver.loan.pbd_hio_loan_account` |
| Affected Column(s) | N/A (empty) |
| Confidence | 0.90 |

**Description:** Two empty tables with `pbd_` prefix exist in the Silver loan schema. `pbd_` does not follow Silver entity naming conventions. These appear to be staging tables for pre-build data (possibly C3 and HIO loan sub-portfolios). Staging tables belong in a staging schema, not in the Silver layer. No logical model entities correspond to these tables.

**Remediation:** Determine purpose of these tables. If staging: relocate to `stg_` prefix in a staging schema. If they represent distinct loan sub-portfolio entities, create corresponding logical model entries with proper names (`c3_loan_account`, `hio_loan_account`) and implement full Silver standards. If obsolete: drop.

**Estimated Effort:** Small
**Owner:** ETL + Governance

---

## 5. Denormalization Register

| # | Co-located Attributes | Entity | Classification | Justification Required? |
|---|---|---|---|---|
| 1 | `Segment`, `Segment Description` in `Loan_Transaction` | loan_transaction | **Unnecessary** | Segment is a Party attribute; should be removed |
| 2 | `Customer number (CIF)` in `Loan_Account` | loan_account_master | **Necessary** | FK reference to Party SA; acceptable as BK |
| 3 | `Product Code`, `Account product id` in `Loan_Account` | loan_account_master | **Acceptable** | Product BK retained for lineage; should FK to `silver.product` |
| 4 | `Branch code` in `Loan_Account` | loan_account_master | **Acceptable** | Branch BK retained; should FK to `silver.org` |
| 5 | `Collection status code/description` in `Loan_Account` | loan_account_master | **Unnecessary** | Should be separate `loan_collection_status` entity with SCD-2 |
| 6 | `Account Score Value`, `Model id`, `Model run id` in `Loan_Account` | loan_account_master | **Unnecessary** | Credit scoring should be separate entity |
| 7 | Tuition loan fields (course, faculty, graduation) in `Loan_Account` | loan_account_master | **Unnecessary** | Product-specific attributes; use companion table or product-type sub-entity |
| 8 | Car loan fields (car status code/description/start date) in `Loan_Account` | loan_account_master | **Unnecessary** | Product-specific; same as above |
| 9 | ESG indicators in `Loan_Account` | loan_account_master | **Unnecessary** | ESG classification is a separate regulatory concern |
| 10 | MTD/YTD/LTD aggregates in `Loan_Account` | loan_account_master | **Unnecessary** — and prohibited | SLV-007 violation; move to Gold |
| 11 | `Collateral Id` in `Loan_Transaction` | loan_transaction | **Unnecessary** | Collateral belongs in `silver.collateral`; FK from loan_account is sufficient |
| 12 | Reference currency and branch data in `silver.loan.*` | reference tables | **Unnecessary** | Should be in `silver.reference` |

---

## 6. Guideline Compliance Summary

| Rule | Rule Text (Quoted) | Status | Entities Affected |
|---|---|---|---|
| **SLV-001** | "`<entity>_sk` (MD5) on every entity" | 🔴 Fail | All entities |
| **SLV-002** | "`<entity>_bk` on every entity" | 🔴 Fail | All entities |
| **SLV-003** | "SCD-2 default" | 🔴 Fail | All entities (no `effective_from`, `effective_to`, `is_current`) |
| **SLV-004** | "No source-system codes; use `silver.reference.code_mapping`" | 🔴 Fail | Status codes, account types mapped directly from source |
| **SLV-005** | "DQ-gated writes; quarantine table" | 🟡 Partial | `loan_account_master` has CDF; no `_dq_status`, no quarantine table visible |
| **SLV-006** | "3NF; no derived values/aggregations" | 🔴 Fail | `Loan_Account` god-table; MTD/YTD/LTD aggregates |
| **SLV-007** | "No pre-computed metrics" | 🔴 Fail | 9 period-aggregate columns in `Loan_Account` |
| **SLV-008** | "All metadata columns present" | 🔴 Fail | All 11 mandatory audit columns absent |
| **SLV-009** | "All timestamps UTC" | 🟡 Unknown | All date columns typed STRING; UTC compliance unverifiable |
| **SLV-010** | "Monetary amounts in smallest unit (fils for AED)" | 🔴 Fail | All monetary amounts typed STRING; no fils conversion |
| **NAM-001** | "snake_case, lowercase, no reserved words" | 🔴 Fail | ~40 columns PascalCase/SCREAMING_CASE in `loan_account_master` |
| **NAM-003** | "Silver entity: `<entity>`" | 🔴 Fail | Active table named `loan_account_master` not `loan_account` |
| **§2.1 (3NF)** | "Decompose entities into 3NF" | 🔴 Fail | `Loan_Account` 141-attribute god-table |
| **§2.3 (SCD-2)** | "Every Silver entity table must have `effective_from`, `effective_to`, `is_current`" | 🔴 Fail | Not present; proxied with CHAR(18) business date columns |
| **§2.4 (SK)** | "Derive surrogate keys deterministically: `MD5(UPPER(TRIM(business_key)))`" | 🔴 Fail | No surrogate keys |
| **§2.6 (No aggregates)** | "Store only atomic, un-aggregated facts in Silver" | 🔴 Fail | MTD/YTD/LTD columns |
| **§2.7 (Audit cols)** | "Include the following Silver technical audit columns on every Silver entity table" | 🔴 Fail | Absent on all tables |
| **§2.9 (UTC)** | "Store all timestamps in UTC" | 🟡 Unknown | Cannot verify since dates are STRING |
| **§2.11 (Model first)** | "Tables that exist in the platform but have no corresponding entity in Erwin model are a governance gap" | 🔴 Fail | LGD tables (3), PBD tables (2) are orphans |
| **§2.12 (Catalog)** | "Register every Silver table in the data catalog" | 🔴 Fail | `description: null` on all 12 physical tables |
| **DQ Gate 2 (Currency)** | "All monetary columns must use `DECIMAL(18,4)`" | 🔴 Fail | All monetary columns are STRING |
| **DQ Gate 2 (Dates)** | "All date columns must conform to `YYYY-MM-DD` format" | 🔴 Fail | All date columns are STRING |

---

## 7. Remediation Plan

### Priority Schedule

#### P0 — Immediate (Sprint 1, Weeks 1–3)

| Action | Finding Ref | Effort | Owner |
|---|---|---|---|
| Add all 11 mandatory audit columns to all physical tables; rename `Business Start Date`/`Business End Date`/`IsLogicallyDeleted` | LoanAccount-006 | Large | Data Modeling + DBA |
| Add `loan_account_sk` (MD5) and `loan_account_bk` to all entities | LoanAccount-005 | Large | Data Modeling + DBA |
| Fix all data types: monetary → `DECIMAL(18,4)`, dates → `DATE`/`TIMESTAMP` | LoanAccount-002, -003 | Large | Data Modeling |
| Create DDL for `loan_agreement` physical table | LoanAgreement-001 | Medium | DBA + ETL |
| Create DDL for `loan_instalment_schedule` physical table + mapping | LoanInstalmentSchedule-001 | Large | Data Modeling + ETL |
| Create DDL for `ifrs9_ecl_staging` physical table + mapping | IFRS9-001 | Large | Data Modeling + Risk + ETL |
| Remove MTD/YTD/LTD aggregates from `Loan_Account` / `loan_account_master` | LoanAccount-004 | Medium | Data Modeling + ETL |

#### P1 — Next Sprint (Sprint 2, Weeks 4–6)

| Action | Finding Ref | Effort | Owner |
|---|---|---|---|
| Rename `loan_account_master` → `loan_account` (breaking change migration) | LoanAccount-008 | Medium | DBA + ETL |
| Decompose `Loan_Account` god-table: extract collection status, credit score, disbursement entities | LoanAccount-001 | Large | Data Modeling |
| Fix column naming: PascalCase → snake_case lowercase | LoanAccount-009 | Medium | Data Modeling |
| Move `reference_*` tables from `silver.loan` to `silver.reference` | Reference-001 | Medium | DBA + ETL |
| Move `Credit_Card_Statement*` entities to Card SA logical model | CardStatement-001 | Small | Data Modeling |
| Fix `Account Cmcy Original Loan Amount` Flexcube mapping semantic error | LoanTermAccount-001 | Small | ETL |
| Resolve `Account number` ambiguity (AGREEMENTID vs. APPLICATION_NUM) | LoanAccount-007 | Medium | Data Modeling + ETL + Governance |
| Create DDL for `loan_statement` and `loan_statement_line_item` | LoanStatement-001 | Large | DBA + ETL |
| Remove `Segment`/`Segment Description` from `loan_transaction` | LoanTransaction-002 | Small | Data Modeling + ETL |
| Create logical model entries for LGD tables; add audit columns | LGD-001 | Medium | Data Modeling + Governance |
| Resolve `pbd_c3_loan` and `pbd_hio_loan_account` — deprecate or rename | PBD-001 | Small | ETL + Governance |
| Clarify `loan_transaction` SA boundary with Transaction SA steward | LoanTransaction-001 | Small | Governance |

#### P2 — Backlog (Sprint 3+)

| Action | Finding Ref | Effort | Owner |
|---|---|---|---|
| Create DDL for `loan_provision_balance_history` and `loan_provision_change_history` | LoanProvision-001 | Large | DBA + ETL + Risk |
| Create DDL for `loan_transaction_account` | — | Medium | DBA + ETL |
| Implement `silver.reference.code_mapping` entries for all loan status/type codes | SLV-004 | Medium | Reference Data team |
| Add table descriptions to all 12 physical tables in data catalog | §2.12 | Small | Governance |
| Add partition columns to tables with large size (`loan_account_master`, `loan_term_account`) | — | Medium | DBA + ETL |
| Implement DQ quarantine table for loan entities | SLV-005 | Medium | ETL |
| Map remaining unmapped Loan_Account attributes (ESG fields — 4 unmapped) | — | Medium | ETL + Data team |
| Implement `loan_disbursement` entity (atomic disbursement events) | LoanAccount-001 | Large | Data Modeling + ETL |
| Implement `loan_esg_indicator` entity | LoanAccount-001 | Medium | Data Modeling + ETL |

---

## 8. Appendix

### A: Mapping Coverage Summary

| Entity | Total Attributes | Mapped | Pending | Not Available | Not Applicable | Coverage % |
|---|---|---|---|---|---|---|
| Loan_Account | 144 | 125 | 9 | 8 | 2 | 87% |
| Loan_Transaction | 43 | 37 | 4 | 2 | 0 | 86% |
| Loan_Term_Account | 14 | 14 | 0 | 0 | 0 | 100% |
| Credit_Agreement | 13 | 12 | 0 | 1 | 0 | 92% |
| Loan_Provision_Balance_History | 7 | 5 | 0 | 2 | 0 | 71% |
| Loan_Provision_Change_History | 8 | 8 | 0 | 0 | 0 | 100% |
| Loan_Instalment_Schedule | 18 | 0 | 0 | 0 | 0 | 0% |
| Loan_Statement | 31 | 0 | 0 | 0 | 0 | 0% |
| Loan_Statement_Line_Item | 21 | 0 | 0 | 0 | 0 | 0% |
| 20× Reference entities | ~155 | ~100 | — | — | — | ~65% |

**Sources:** RLS (Intellect/LEAM), Flexcube (FCUBSPROD). No Temenos/other sources evident.

### B: Guideline Citations

| Citation | Rule |
|---|---|
| `guidelines/11-modeling-dos-and-donts.md §2.1` | Third Normal Form — no repeating groups, no transitive dependencies |
| `guidelines/11-modeling-dos-and-donts.md §2.3` | SCD Type 2 history |
| `guidelines/11-modeling-dos-and-donts.md §2.4` | Deterministic MD5 surrogate keys |
| `guidelines/11-modeling-dos-and-donts.md §2.5` | Business key and source code canonicalization |
| `guidelines/11-modeling-dos-and-donts.md §2.6` | No aggregations or derived metrics in Silver |
| `guidelines/11-modeling-dos-and-donts.md §2.7` | Mandatory Silver technical audit columns (11 columns) |
| `guidelines/11-modeling-dos-and-donts.md §2.9` | UTC timestamps only |
| `guidelines/11-modeling-dos-and-donts.md §2.11` | Erwin model before DDL |
| `guidelines/11-modeling-dos-and-donts.md §2.12` | Informatica GDGC registration |
| `guidelines/06-naming-conventions.md §NAM-001` | snake_case lowercase identifiers |
| `guidelines/06-naming-conventions.md §NAM-003` | Silver entity table naming: `<entity>` |
| `guidelines/01-data-quality-controls.md §Gate 2` | Monetary columns `DECIMAL(18,4)`, dates `YYYY-MM-DD`, error rate threshold 5% |
| `guidelines/01-data-quality-controls.md §5` | Mandatory audit columns: 11 columns with canonical names |

### C: Industry References

| Reference | Applicability |
|---|---|
| **BIAN v11 — Loan service domain** | Canonical entity boundaries for loan account, loan agreement, repayment schedule |
| **IFRS 9 (IASB)** | ECL staging, provision entities, LGD/PD/EAD measurement requirements |
| **BCBS 239 Principle 3 (Accuracy)** | Requires complete, accurate credit risk data; missing provision and IFRS 9 entities are direct violations |
| **CBUAE Circular on IFRS 9 (2018)** | UAE-specific ECL staging guidance requiring Stage 1/2/3 classification with documented staging criteria |
| **IFW (IBM/Teradata)** | Party-Role-Account pattern; Loan Account as a Financial Account sub-type |
| **Kimball Group** | Prohibition on pre-aggregated metrics in the integration layer (Silver equivalent) |

---

### Summary Score Card

| Category | Score | Notes |
|---|---|---|
| Physical Completeness | 3/10 | 5 critical entities missing physical tables; 5 tables empty; 3 orphan active tables |
| Metadata Compliance | 1/10 | All 11 mandatory audit columns absent; non-standard proxies used |
| Key Strategy | 0/10 | Zero surrogate or business keys defined |
| Data Typing | 0/10 | All 598 attributes STRING; monetary/date types incorrect throughout |
| 3NF Compliance | 2/10 | 141-attribute god-table; 10 unnecessary denormalizations identified |
| Naming Conventions | 3/10 | Active table has wrong name; ~40 columns non-compliant |
| Mapping Coverage | 6/10 | 86-100% on mapped entities; 0% on 3 critical entities |
| IFRS 9 / Regulatory | 1/10 | ECL staging entirely absent; provision tables unmapped |
| **Overall** | **2/10** | **Not production-ready; requires significant remediation before Silver promotion** |