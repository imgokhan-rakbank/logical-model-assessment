# Credit Card — Silver Layer Gap Assessment Report

| Field | Value |
|---|---|
| **Subject Area** | Credit Card |
| **Canonical SA Name** | Card (`silver.card`) per `sa/subject-areas.md`; physical schema `silver_dev.credit_card` |
| **SA Directory** | `sa/credit-card/` |
| **Assessment Date** | 2026-04-27 |
| **Overall State** | 🔴 **Incorrect** — Structural, regulatory, and data-loss risks identified across all core entities |
| **Assessor** | Automated Gap Assessment — LLM Senior Data Architect |
| **Logical Entities Assessed** | 50 (7 core, 43 reference) |
| **Physical Tables Found** | 37 (including 3 temp/test tables; 2 core entities have no physical table) |
| **Total Findings** | 48 |
| **P0 Findings** | 12 |
| **P1 Findings** | 22 |
| **P2 Findings** | 14 |
| **Overall Confidence** | 0.92 |

---

## 1. Executive Summary

### 1A — Management Slide

**One-Line Health**: The Credit Card silver subject area is **not production-ready** — it carries critical structural defects spanning all key entities, with 12 P0 blockers requiring immediate remediation before any Gold or regulatory consumer can safely rely on this data.

**Top 5 Risks**

1. 🔴 **All attributes typed as STRING** — Monetary amounts, dates, booleans, and identifiers are uniformly `STRING`; no precision, no constraint enforcement. Downstream calculations will silently corrupt financial figures.
2. 🔴 **No surrogate or business keys** — Not a single entity has an `<entity>_sk` (MD5 surrogate) or `<entity>_bk` (business key). SCD-2 history is broken; Gold FK joins will fail.
3. 🔴 **Credit_Card_Invoice fully unmapped** — 100% of invoice attributes have `Status = Not Available`. No physical table exists. The billing lifecycle for revolving credit is missing entirely.
4. 🔴 **Temp/test tables in production schema** — `credit_card_account_temp`, `credit_card_account_test2703`, `credit_card_event11` exist in `silver_dev.credit_card`, contaminating the authoritative layer.
5. 🔴 **Repeating-group columns violate 3NF** — `Amount in arrears 1–8 period` and `Fee table 1–2` in `Credit_Card_Account` are unnormalised repeating groups; analysis across arrears buckets requires UNPIVOT, breaks indexing, and will silently drop data when a ninth bucket is added at source.

**Top 5 Priority Actions**

| Priority | Action | Owner | Effort | Deadline |
|---|---|---|---|---|
| P0-1 | Introduce `credit_card_account_sk`, `credit_card_account_bk` (MD5) and SCD-2 audit columns on all core entities | Data Modeling | Large | Sprint 1 |
| P0-2 | Retype all monetary amounts to `DECIMAL(18,4)`, dates to `DATE`, timestamps to `TIMESTAMP`, booleans to `BOOLEAN` | Data Modeling + ETL | Large | Sprint 1 |
| P0-3 | Drop `credit_card_account_temp`, `credit_card_account_test2703`, `credit_card_event11` from `silver_dev.credit_card` | DBA | Small | Sprint 1 |
| P0-4 | Design and map `Credit_Card_Invoice` entity; create physical table `silver.credit_card.credit_card_invoice` | Data Modeling + ETL | Large | Sprint 2 |
| P0-5 | Normalize `Amount in arrears 1–8` repeating group into `credit_card_arrears_bucket` child table | Data Modeling | Medium | Sprint 2 |

**KPIs (Target State)**

- Mapping coverage: 85%+ per core entity (current: Invoice 0%, Account 80%, Statement 74%)
- SLV-008 metadata compliance: 100% of core Silver tables (current: ~0%)

**Decision Recommendation**: Do NOT promote Credit Card Silver to Gold consumption until P0 blockers are resolved. Escalate to Group Data Office for emergency remediation sprint resourcing.

### 1B — Technical Summary

The Credit Card subject area (`silver_dev.credit_card`) contains 37 physical Delta tables built against a 50-entity logical model in the `CreditCard` subject area. The subject area taxonomy (`sa/subject-areas.md`) defines this domain as "Card" under `silver.card` (Priority 2, High) — a naming divergence from both the physical schema (`credit_card`) and the directory (`sa/credit-card`) that creates governance ambiguity.

Seven core transactional entities are modeled: `Credit_Card_Account`, `Credit_Card_Transaction`, `Credit_Card_Statement`, `Credit_Card_Statement_Line_Item`, `Credit_Card_Payment_Schedule`, `Credit_Card_Receivables`, and `Credit_Card_Invoice`. Two (Invoice and Statement) have no deployed physical tables. The remaining five have deployed tables but suffer from uniform `STRING` typing across all attributes, absent surrogate/business key columns, missing SCD-2 metadata, and multiple 3NF violations. The `Loan_Instalment_Schedule` entity is incorrectly scoped inside this SA (belongs to `silver.lending`). Reference entities for Exchange Rate, Currency, and Bank Branch belong to `silver.reference` and `silver.org` respectively.

Mapping completeness is variable: the Transaction entity is well-mapped (96%), Account is partially mapped (80%), and Invoice is entirely unmapped (0%). Two sources — PRIME/TCTDBS and FLEXUAT/FCUBSPROD — feed most entities without a multi-source companion table pattern, risking silent attribute overwrite. Source-system codes are mapped directly to Silver columns without canonicalization through `silver.reference.code_mapping`.

---

## 2. Subject Area Architecture Assessment

### 2.1 Domain Scoping & BIAN Alignment

| Dimension | Assessment | Detail |
|---|---|---|
| **BIAN Alignment** | 🟡 Partial | The subject area spans BIAN *Credit Card* (v11 §5.4) and partially overlaps *Collections* and *Payments*. `Credit_Card_Receivables` models collection activity that belongs in `silver.collection` (BIAN *Collections*). The `Reference_event` entity duplicates the Business Event domain. |
| **SA Taxonomy Match** | 🔴 Incorrect | `sa/subject-areas.md` names this domain **"Card"** with schema `silver.card` and describes card-level plastic/revolving details. The physical implementation uses `silver_dev.credit_card` (different schema name). This divergence will cause issues in catalog registration and cross-SA FK references. |
| **Directory Naming** | 🔴 Incorrect | Directory is `sa/credit-card/` (kebab-case). All other SA directories use underscore (e.g., `sa/customer_risk_rating/`). Subject area references should consistently use `credit_card`. |
| **Scope Overlap** | 🔴 Incorrect | 14 of 50 entities belong to other SAs: `Loan_Instalment_Schedule` → `silver.lending`; `Reference_Exchange_Rate`, `Reference_Currency`, `Reference_Currency_Code` → `silver.reference`; `Reference_Bank_Branch` → `silver.org`; `Reference_ESG_Status` → `silver.risk`; `Reference_Loan_Status` → `silver.lending`; `Reference_event` → `silver.event`. |
| **Missing BIAN Entities** | 🔴 Incorrect | Card Authorization (BIAN *Card Authorization*) entity absent; Card Audit Log absent (specifically called out in SA taxonomy description); Card Plastic / Token entity absent. |

### 2.2 Identity Strategy

| Rule | Status | Evidence |
|---|---|---|
| `<entity>_sk` (MD5 surrogate) on every entity | 🔴 Absent | Zero physical or logical entities carry a `_sk` column. |
| `<entity>_bk` (business key) on every entity | 🔴 Absent | No `_bk` columns. Business keys exist only under descriptive names (`Account Number`, `Transaction Id`) with no standard naming or non-null constraint. |
| Deterministic MD5 derivation | 🔴 N/A | Cannot be verified — no SK exists. |
| PK uniqueness constraint | 🔴 Absent | No PK constraints visible in physical table registry. |

### 2.3 SCD Strategy

| Rule | Status | Evidence |
|---|---|---|
| SCD-2 default on all entities | 🔴 Absent | Logical model carries `Business Start Date` (CHAR(18)), `Business End Date` (CHAR(18)), `Is Active Flag` (CHAR(18)). These do not conform to the guideline standard column names or types (`effective_from TIMESTAMP`, `effective_to TIMESTAMP`, `is_current BOOLEAN`). |
| `effective_from` / `effective_to` / `is_current` | 🔴 Wrong type | All three SCD columns are typed `CHAR(18)` in the logical model instead of TIMESTAMP/BOOLEAN. |
| `source_system_code`, `create_date`, `update_date` | 🟡 Present but typed STRING | Audit lineage columns exist as STRING in the logical model; `delete_date` is mapped as `Deleted Date`. Missing: `run_id`, `source_ingestion_date`. |

### 2.4 Cross-Entity Relationships & Cardinality

| Relationship | Expected Cardinality | Status |
|---|---|---|
| `Credit_Card_Account` → Customer (Party) | M:1 | 🔴 `Customer Number (CIF)` is embedded as a plain STRING attribute with no FK declaration or SK reference to `silver.party`. |
| `Credit_Card_Transaction` → `Credit_Card_Account` | M:1 | 🔴 `Account Number` embedded as STRING; no SK FK. |
| `Credit_Card_Statement` → `Credit_Card_Account` | M:1 | 🔴 Same issue. |
| `Credit_Card_Statement_Line_Item` → `Credit_Card_Statement` | M:1 | 🔴 `Statement Document ID` present but no FK constraint or SK join. |
| `Credit_Card_Receivables` → `Credit_Card_Payment_Schedule` | M:1 | 🟡 `Payment Schedule ID` present; SK still absent. |
| `Credit_Card_Payment_Schedule` → `Credit_Card_Invoice` | M:1 | 🔴 `Invoice` entity has no physical table; FK is broken. |

### 2.5 ETL Mapping & Lineage Completeness

| Entity | Logical Attrs | Mapping Rows | Mapped | Not Available | Coverage |
|---|---|---|---|---|---|
| `Credit_Card_Account` | 79 | 74 | 59 | 15 | 75% |
| `Credit_Card_Transaction` | 49 | 51 | 49 | 2 | 96% |
| `Credit_Card_Statement` | 31 | 23 | 22 | 1 | 71% |
| `Credit_Card_Statement_Line_Item` | 26 | 9 | 7 | 2 | 27% |
| `Credit_Card_Payment_Schedule` | 21 | 4 | 4 | 0 | 19% |
| `Credit_Card_Receivables` | 20 | 9 | 8 | 1 | 40% |
| `Credit_Card_Invoice` | 22 | 11 | 0 | 11 | **0%** |

**Two distinct source systems** (PRIME/TCTDBS and FLEXUAT/FCUBSPROD) feed `Credit_Card_Account` and `Credit_Card_Transaction` without a multi-source companion table pattern — the mapping assigns alternate values per column rather than managing them as parallel source rows.

### 2.6 Major Design Gaps

| # | Gap | Severity |
|---|---|---|
| G-01 | No SK/BK identity strategy implemented on any entity | P0 Critical |
| G-02 | All attributes typed STRING — no monetary precision, date enforcement, boolean safety | P0 Critical |
| G-03 | SCD-2 metadata columns wrong type (CHAR(18)) and missing columns (`run_id`, `source_ingestion_date`) | P0 Critical |
| G-04 | `Credit_Card_Invoice` entity has no physical table and 0% mapping coverage | P0 Critical |
| G-05 | Temp/test tables (`_temp`, `_test2703`, `event11`) in production Silver schema | P0 Critical |
| G-06 | `Loan_Instalment_Schedule` misplaced in Credit Card SA | P1 High |
| G-07 | 14 reference/cross-domain entities belong in other SAs | P1 High |
| G-08 | `Amount in arrears 1–8` repeating group (3NF violation) in `Credit_Card_Account` | P0 Critical |
| G-09 | `Credit_Card_Statement` contains aggregate metrics (Total Purchase Amount, Cash Advance Count) — SLV-007 violation | P1 High |
| G-10 | No canonicalization of source codes via `silver.reference.code_mapping` | P1 High |
| G-11 | Subject area canonical name/schema divergence (`Card`/`silver.card` vs `credit_card`/`silver_dev.credit_card`) | P1 High |
| G-12 | No card authorization or card audit log entity — called out explicitly in SA taxonomy | P1 High |

---

## 3. Entity Inventory

| # | Entity | State | Criticality | Physical Table | Unmapped Attrs | Findings | Confidence |
|---|---|---|---|---|---|---|---|
| 1 | `Credit_Card_Account` | 🔴 Incorrect | High | `credit_card_account` | 15 | 9 | 0.93 |
| 2 | `Credit_Card_Transaction` | 🔴 Incorrect | High | `credit_card_transaction` | 2 | 7 | 0.95 |
| 3 | `Credit_Card_Statement` | 🔴 Incorrect | High | *(missing)*  | 1 | 6 | 0.92 |
| 4 | `Credit_Card_Statement_Line_Item` | 🔴 Incorrect | High | `credit_card_statement_line_item` | 19 | 5 | 0.88 |
| 5 | `Credit_Card_Payment_Schedule` | 🔴 Incorrect | High | `credit_card_payment_schedule` | 17 | 5 | 0.90 |
| 6 | `Credit_Card_Receivables` | 🟡 Partial | Medium | `credit_card_receivables` | 12 | 4 | 0.88 |
| 7 | `Credit_Card_Invoice` | 🔴 Incorrect | High | *(missing)* | 22 | 4 | 0.98 |
| 8 | `Credit_Card_event` *(mapping only)* | 🟡 Partial | Medium | `credit_card_event` | N/A | 3 | 0.82 |
| 9 | `Loan_Instalment_Schedule` | 🔴 Incorrect | High | *(no match)* | All | 2 | 0.97 |
| 10 | `Reference_*` (39 entities) | 🟡 Partial | Low–Med | Various `reference_*` | Various | 9 | 0.85 |

---

## 4. Per-Entity Assessments

### 4.1 Credit_Card_Account

**State**: 🔴 Incorrect | **Criticality**: High | **Confidence**: 0.93
**Physical Table**: `silver_dev.credit_card.credit_card_account`
**Sources**: PRIME/TCTDBS (primary), CAPS/CAPSADMIN (secondary), FLEXUAT/FCUBSPROD (tertiary)

#### 4.1.1 Industry Fitness

`Credit_Card_Account` is the central entity of the subject area, representing the revolving credit agreement between the bank and a customer. The entity is over-wide (79 attributes), mixing account meta-data, balance snapshots, interest rates, arrears history (buckets 1–8), fee tables, segment codes, status/blocking information, and regulatory classification. Per BIAN, these concerns should be separated across at minimum: *Credit Card Account* (terms and status), *Credit Card Balance* (balance snapshot — best in `silver.account`), and *Credit Card Arrears* (delinquency history). The IFW Banking Data Model similarly separates `FINANCIAL_ACCOUNT` from `ACCOUNT_BALANCE` and `DELINQUENCY_EVENT`.

The entity has **8 arrears bucket columns** (`Amount in arrears 1–8 period`) which are a repeating group violating 1NF/3NF. Multiple credit limit columns (`Account permanent Credit Limit`, `Account Temporary Credit Limit`, `Account Credit Limit`, `Permanent credit limit`, `Current credit limit`) indicate poor normalization — at least 3 of these are likely duplicates or derived from each other, requiring deduplication. Three interest rate columns (`Interest Rate`, `Cash Advance Interest Rate`, `Retail interest rate`, `Robinson Rate`, `Balance Transfer rate`) appear to represent different rate types and should be normalised into a `card_interest_rate` child table keyed by `rate_type_code`.

#### 4.1.2 Attribute-Level Review

| Attribute | LM Type | Expected Type | Mapping | Finding |
|---|---|---|---|---|
| Account number | STRING | STRING | PRIME.TCTDBS.CACCOUNTS.SERNO | Missing `_bk` suffix; should be `credit_card_account_bk` |
| *credit_card_account_sk* | *(absent)* | STRING NOT NULL | N/A | **MISSING — P0** |
| *credit_card_account_bk* | *(absent)* | STRING NOT NULL | N/A | **MISSING — P0** |
| Account type | STRING | STRING (code) | CACCOUNTS.ACCOUNTTYPE | Should canonicalize via `code_mapping`; raw CHAR(1) stored as-is |
| Account open date | STRING | DATE | CREATEDATE (DATE) | Type mismatch: source is DATE, target is STRING |
| Account close date | STRING | DATE | CLOSEDATE (DATE) | Type mismatch |
| Contract type | STRING | STRING (code) | Not Available (FLEXUAT only) | Partial mapping; not available in primary source |
| Account currency | STRING | CHAR(3) | CACCOUNTS.CURRENCY | Should be CHAR(3) ISO 4217 |
| Secured indicator | STRING | BOOLEAN | Not Available | Unmapped; should be `is_secured` BOOLEAN |
| Temporary credit limit | STRING | DECIMAL(18,4) | Not Available | Unmapped monetary; should be `temp_credit_limit_amount_aed` DECIMAL(18,4) |
| Customer number (CIF) | STRING | STRING (FK → party) | Multiple sources | Should be `customer_bk` STRING NOT NULL, FK to `silver.party.customer` |
| Over limit excess amount | STRING | DECIMAL(18,4) | CUROVERLIMITAMOUNT (FLOAT) | Monetary stored as STRING; source FLOAT; should be DECIMAL(18,4) |
| Account permanent credit limit amount | STRING | DECIMAL(18,4) | PERMANENTLIMIT (FLOAT) | Monetary stored as STRING |
| Account credit limit | STRING | DECIMAL(18,4) | CREDITLIMIT (NUMBER(22)) | Monetary stored as STRING; duplicate of `Permanent credit limit` — deduplication required |
| Permanent credit limit | STRING | DECIMAL(18,4) | PERMANENTLIMIT (FLOAT) | Same source as `Account permanent credit limit amount` — **duplicate mapping** |
| Current credit limit | STRING | DECIMAL(18,4) | ACCOUNTS.CREDITLIMIT | Monetary stored as STRING |
| Behavior score | STRING | INTEGER | APPLICATIONS.SCORE (NUMBER) | Numeric score stored as STRING |
| Current account balance | STRING | DECIMAL(18,4) | CACCOUNTS.BALANCE (NUMBER(16,3)) | Monetary stored as STRING; source has precision (16,3) |
| Interest rate | STRING | DECIMAL(10,6) | CUROUTBALINTEREST (FLOAT(126)) | Percentage stored as STRING; precision lost |
| Cash advance interest rate | STRING | DECIMAL(10,6) | CUROUTBALINTEREST (FLOAT(126)) | **Same source column as `Interest rate`** — ambiguous mapping |
| Amount in arrears 1–8 period | STRING ×8 | DECIMAL(18,4) ×8 | BUCKET1–8 | Repeating group — 3NF violation; all 8 buckets map individually |
| Latest payment date | STRING | DATE | lastpayrecvdate (DATE) | Type mismatch |
| Balance brought forward | STRING | DECIMAL(18,4) | curoutbaltotal (Number) | Monetary stored as STRING |
| Agreement status | STRING | STRING (code) | GENERALSTAT (String) | Source-system code stored without canonicalization |
| Date change of agreement status | STRING | DATE | GENERALSTATDATE (DATE) | Type mismatch |
| Monitor status code | STRING | STRING (code) | CARDSTATUS.BLOCK_STATUS | Source-system code stored without canonicalization |
| Block status code | STRING | STRING (code) | ACCOUNTS.BLACKLIST (NUMBER) | Numeric code stored as STRING |
| Account status code | STRING | STRING (code) | CARDSTATUS.CODE (CHAR) | Source-system code stored without canonicalization |
| Account status start date | STRING | DATE | MAINTAINDATE (DATE) | Type mismatch |
| Threshold amount | STRING | DECIMAL(18,4) | maxcreditlimit (FLOAT) | Monetary stored as STRING |
| Amount due | STRING | DECIMAL(18,4) | OVERDUEAMOUNT (NUMBER) | Monetary stored as STRING |
| Latest due date | STRING | DATE | MINAMTDUEDATE (DATE) | Type mismatch |
| Payoff flag | STRING | BOOLEAN | PAYMENTINDICATOR (NUMBER) | Flag stored as STRING; should be `is_paid_off` BOOLEAN |
| Fee table 1 | STRING | (child table) | Not Available | Repeating group — should be separate `card_fee_schedule` entity |
| Fee table 2 | STRING | (child table) | Not Available | Repeating group |
| Regulators classification code | STRING | STRING (code) | Not Available | Unmapped regulatory field |
| Business Start Date | CHAR(18) | TIMESTAMP | — | Wrong type: CHAR(18) should be TIMESTAMP (SCD `effective_from`) |
| Business End Date | CHAR(18) | TIMESTAMP | — | Wrong type: CHAR(18) should be TIMESTAMP (SCD `effective_to`) |
| Is Active Flag | CHAR(18) | BOOLEAN | — | Wrong type: CHAR(18) should be BOOLEAN (`is_current`) |
| Source system code | STRING | STRING | — | Correct column name but no `run_id` or `source_ingestion_date` present |

#### 4.1.3 Metadata Completeness

| Column | Required Type | Status |
|---|---|---|
| `source_system_code` | STRING | 🟢 Present (as `Source system code`) |
| `source_system_id` | STRING | 🟢 Present |
| `create_date` | TIMESTAMP | 🟡 Present as STRING |
| `update_date` | TIMESTAMP | 🟡 Present as STRING |
| `delete_date` | TIMESTAMP | 🟡 Present as `Deleted date` STRING |
| `is_active_flag` | STRING | 🟡 Present as CHAR(18) |
| `effective_from` | TIMESTAMP | 🔴 Missing — `Business Start Date` CHAR(18) is not a valid substitute |
| `effective_to` | TIMESTAMP | 🔴 Missing — `Business End Date` CHAR(18) is not a valid substitute |
| `is_current` | BOOLEAN | 🔴 Missing — `Is Active Flag` CHAR(18) is not a valid substitute |
| `run_id` | STRING | 🔴 Missing |
| `source_ingestion_date` | TIMESTAMP | 🔴 Missing |

---

#### Findings

### Finding CCA-001 — Missing Surrogate Key and Business Key

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | `SLV-001` — "Every Silver entity must have an `<entity>_sk` (MD5 surrogate key) column" and `SLV-002` — "Every Silver entity must have an `<entity>_bk` (business key) column" |
| Evidence | Logical model: `Credit_Card_Account` has no `_sk` or `_bk` attribute · Physical: `credit_card_account` table has no `credit_card_account_sk` or `credit_card_account_bk` |
| Affected Table | `silver_dev.credit_card.credit_card_account` |
| Affected Column(s) | `credit_card_account_sk` (missing), `credit_card_account_bk` (missing) |
| Confidence | 0.99 |

**Description:** The entity has no surrogate key and no standardized business key column. Without `credit_card_account_sk`, Gold FK joins cannot be built, and SCD-2 history cannot be tracked. The `Account Number` attribute from PRIME.CACCOUNTS.SERNO is the candidate business key but is unnamed as such and carries no NOT NULL constraint.

**Remediation:** Add `credit_card_account_sk STRING NOT NULL` (MD5 of `credit_card_account_bk`) and `credit_card_account_bk STRING NOT NULL` (sourced from CACCOUNTS.SERNO). Enforce NOT NULL + UNIQUE. Update all downstream Gold pipelines.

**Estimated Effort:** Large
**Owner:** Data Modeling + ETL

---

### Finding CCA-002 — All Attributes Typed STRING (Monetary, Date, Boolean Precision Lost)

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | `SLV-010` — "Monetary amounts must be stored in smallest unit (fils) using BIGINT or DECIMAL(18,4)" and Gate 2 standard — "All monetary columns must use DECIMAL(18,4)" |
| Evidence | Logical model: all 79 attributes typed STRING · Data mapping: source columns are NUMBER(16,3), FLOAT(126), DATE, CHAR — none mapped to their canonical types |
| Affected Table | `silver_dev.credit_card.credit_card_account` |
| Affected Column(s) | All monetary amounts (≥15 columns), all date columns (≥10 columns), `Payoff flag`, `Secured indicator` |
| Confidence | 0.99 |

**Description:** Every attribute in the logical model is typed `STRING`. Source columns include NUMBER(16,3) for balances, FLOAT(126) for interest rates, DATE for open/close dates, and CHAR(1) for type codes. Storing these as STRING: (1) silently truncates decimal precision, (2) prevents arithmetic operations, (3) bypasses range validation, and (4) inflates storage. This violates Gate 2 data quality requirements and SLV-010.

**Remediation:** Retype per canonical standards: monetary fields → `DECIMAL(18,4)` (amounts in AED; note: per UAE convention, 1 AED = 100 fils); date fields → `DATE`; timestamp fields → `TIMESTAMP`; boolean indicators → `BOOLEAN`; code fields → `STRING` (with length constraints where appropriate).

**Estimated Effort:** Large
**Owner:** Data Modeling + ETL

---

### Finding CCA-003 — Repeating Group: Amount in Arrears Buckets 1–8 (3NF Violation)

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | `SLV-006` — "3NF; no derived values/aggregations"; modeling guideline §2.1 — "Repeating groups (sets of similar attributes that recur) must become child tables" |
| Evidence | Logical model lines 30–34 and 49–51: `Amount in arrears 1 period` through `Amount in arrears 8 period` (8 identical-structure columns, sourced from BUCKET1–8) |
| Affected Table | `silver_dev.credit_card.credit_card_account` |
| Affected Column(s) | `Amount in arrears 1–8 period` |
| Confidence | 0.98 |

**Description:** Eight columns of identical structure represent past-due amounts by billing cycle bucket. This is a textbook 1NF/3NF violation: (1) queries for "any bucket > 0" require OR across 8 conditions; (2) adding a 9th bucket requires a DDL change; (3) analytics cannot slice by bucket number without UNPIVOT. Source has BUCKET1–BUCKET8 in CAPS.CAPSADMIN.ACCOUNT_DELINQUENCY.

**Remediation:** Create child entity `credit_card_arrears_bucket` with columns: `credit_card_account_sk`, `bucket_number INTEGER NOT NULL`, `arrears_amount_aed DECIMAL(18,4)`, `as_of_date DATE`, plus standard SCD-2 audit columns.

**Estimated Effort:** Medium
**Owner:** Data Modeling

---

### Finding CCA-004 — Duplicate Mapping: Interest Rate and Cash Advance Interest Rate

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | High |
| Guideline Rule | Mapping uniqueness / lineage correctness |
| Evidence | Mapping rows 40–41: `Interest Rate` ← `CUROUTBALINTEREST`; `Cash advance Interest Rate` ← `CUROUTBALINTEREST` (same column) |
| Affected Table | `silver_dev.credit_card.credit_card_account` |
| Affected Column(s) | `interest_rate`, `cash_advance_interest_rate` |
| Confidence | 0.95 |

**Description:** Both the base interest rate and cash advance interest rate are sourced from `CAPS.CAPSADMIN.ACCOUNT_CURR_BALANCES.CUROUTBALINTEREST`. This is either (1) an incorrect mapping (cash advance rate should come from a different source column) or (2) confirms both attributes represent the same rate, in which case one should be removed. This ambiguity will produce incorrect regulatory disclosures if the rates differ in reality.

**Remediation:** Clarify with business SME whether cash advance rate maps to a different CAPS column (e.g., `CUROUTBALCASHTRANS`). Update mapping. Add mapping verification step to ETL pipeline.

**Estimated Effort:** Small
**Owner:** ETL / Business Analyst

---

### Finding CCA-005 — Duplicate Mapping: Permanent Credit Limit (Two Attributes, Same Source)

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | Medium |
| Guideline Rule | `SLV-006` — "No derived values" |
| Evidence | Mapping rows 31 and 34: `Account permanent Credit Limit amount` ← `CARD_GENERAL.PERMANENTLIMIT`; `Permanent credit limit` ← `CARD_GENERAL.PERMANENTLIMIT` (same column) |
| Affected Table | `silver_dev.credit_card.credit_card_account` |
| Affected Column(s) | `account_permanent_credit_limit_amount`, `permanent_credit_limit` |
| Confidence | 0.96 |

**Description:** Two distinct logical attributes map to the identical source column. One must be removed.

**Remediation:** Retain `account_permanent_credit_limit_amount` (follows naming convention `<measure>_amount_<currency>`). Remove `Permanent credit limit` as a duplicate. Confirm with business.

**Estimated Effort:** Small
**Owner:** Data Modeling

---

### Finding CCA-006 — SCD-2 Metadata Columns Wrong Type and Incomplete

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | `SLV-003` — "SCD-2 default"; guideline §2.7 — "Mandatory Silver Technical Audit Columns" |
| Evidence | Logical model attrs 79–81: `Business Start Date CHAR(18)`, `Business End Date CHAR(18)`, `Is Active Flag CHAR(18)` |
| Affected Table | `silver_dev.credit_card.credit_card_account` |
| Affected Column(s) | `Business Start Date`, `Business End Date`, `Is Active Flag`, missing `run_id`, `source_ingestion_date` |
| Confidence | 0.99 |

**Description:** SCD-2 range columns are typed `CHAR(18)` instead of `TIMESTAMP`. A `CHAR(18)` cannot store a proper timestamp — the column will either be unusable for range queries or require implicit casting. `Is Active Flag` must be `BOOLEAN`. Additionally, `run_id` and `source_ingestion_date` are absent from the logical model, breaking pipeline lineage traceability.

**Remediation:** Rename and retype: `Business Start Date` → `effective_from TIMESTAMP NOT NULL`, `Business End Date` → `effective_to TIMESTAMP`, `Is Active Flag` → `is_current BOOLEAN NOT NULL`. Add `run_id STRING` and `source_ingestion_date TIMESTAMP`.

**Estimated Effort:** Medium
**Owner:** Data Modeling + ETL

---

### Finding CCA-007 — Source-System Codes Not Canonicalized (SLV-004)

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | High |
| Guideline Rule | `SLV-004` — "No source-system codes; use `silver.reference.code_mapping`" |
| Evidence | Mapping: `Agreement status` ← `CAPSMAIN.GENERALSTAT`; `Account status code` ← `CARDSTATUS.CODE`; `Block status code` ← `ACCOUNTS.BLACKLIST` (numeric) |
| Affected Table | `silver_dev.credit_card.credit_card_account` |
| Affected Column(s) | `agreement_status`, `account_status_code`, `block_status_code`, `monitor_status_code` |
| Confidence | 0.93 |

**Description:** Status codes are sourced directly from CAPS internal columns (e.g., BLACKLIST is a numeric code, GENERALSTAT is a CBS-specific status string) without canonicalization. Consumers of Silver data will receive opaque codes that differ across source systems.

**Remediation:** Route all status codes through `silver.reference.code_mapping` JOIN on `(source_system='CAPS', source_domain='ACCOUNT_STATUS', source_code=...)` to derive canonical values.

**Estimated Effort:** Medium
**Owner:** ETL

---

### Finding CCA-008 — Regulatory Fields Not Mapped (CBUAE Compliance Risk)

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | BCBS 239 Principle 2 (Data Architecture & IT Infrastructure); CBUAE regulatory reporting requirements |
| Evidence | Mapping: `Regulators classification code` = Not Available; `Regulators classification date` = Not Available; `Regulators classification reason date` = Not Available |
| Affected Table | `silver_dev.credit_card.credit_card_account` |
| Affected Column(s) | `regulators_classification_code`, `regulators_classification_date`, `regulators_classification_reason_date` |
| Confidence | 0.92 |

**Description:** CBUAE credit card regulatory classification (IFRS 9 Stage 1/2/3 equivalent for revolving credit) is entirely unmapped. This data feeds regulatory reporting and IFRS 9 ECL calculations for card portfolios.

**Remediation:** Source from PRIME/TCTDBS or FLEXUAT; confirm source mapping with Compliance team. Prioritise as CDE (Critical Data Element).

**Estimated Effort:** Medium
**Owner:** ETL / Compliance

---

### Finding CCA-009 — Repeating Group: Fee Table Columns

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | Medium |
| Guideline Rule | `SLV-006` — "3NF; no derived values"; modeling guideline §2.1 repeating groups |
| Evidence | Logical model attrs 63–65: `Fee table 1`, `Fee table 2`, `Fee table 2 from valid date` |
| Affected Table | `silver_dev.credit_card.credit_card_account` |
| Affected Column(s) | `fee_table_1`, `fee_table_2`, `fee_table_2_from_valid_date` |
| Confidence | 0.90 |

**Description:** Two fee table reference columns indicate a repeating pattern. Fee schedules should be a separate FK reference to a `card_fee_schedule` entity, not denormalized into the account row.

**Remediation:** Create `card_fee_schedule` child entity. FK from `credit_card_account` to `card_fee_schedule` on `fee_schedule_bk`.

**Estimated Effort:** Medium
**Owner:** Data Modeling

---

### 4.2 Credit_Card_Transaction

**State**: 🔴 Incorrect | **Criticality**: High | **Confidence**: 0.95
**Physical Table**: `silver_dev.credit_card.credit_card_transaction`
**Sources**: PRIME/TCTDBS, FLEXUAT/FCUBSPROD

#### 4.2.1 Industry Fitness

The transaction entity appropriately captures POS, e-commerce, cash advance, and token-based transactions. Grain appears to be one row per transaction authorization/posting. However, the entity conflates authorization, settlement, and posting events in a single row. Per BIAN *Card Transaction* and IFW `FINANCIAL_TRANSACTION`, a separate authorization entity should exist for pre-authorization records. The entity lacks a clear double-entry ledger reference — `Ledger Batch Id` is present but not FK-linked to `silver.gl`.

Card Number is present as a full (masked?) field — PII control must be confirmed. No explicit SHA-256 masking transformation is noted in the mapping.

#### 4.2.2 Attribute-Level Review (Key Issues)

| Attribute | Issue |
|---|---|
| `credit_card_transaction_sk` | **Missing** — no surrogate key |
| `Transaction Amount` | STRING; source NUMBER(22); should be DECIMAL(18,4) |
| `Transaction Date` | STRING; source DATE; should be DATE |
| `Transaction Time` | STRING; source DATE; should be TIMESTAMP |
| `Transaction Posted Date` | STRING; source DATE; should be DATE |
| `Card Number` | STRING; **PII** — SHA-256 masking not confirmed in mapping |
| `Transaction Amount` ↔ `Source Currency Amount` | Both map to `CTRANSACTIONS.I004_AMT_TRXN` — **duplicate/ambiguous** |
| `Transaction Source` ↔ `Transaction Rejection Code If QR` | Both map to `CTRANSACTIONS.ORIGINATOR` — **duplicate mapping** |
| `Deposit Account Segment Code` | Maps to `CTRANSACTIONS.MSGTYPE` — semantic mismatch (MSG_TYPE ≠ deposit segment) |
| `Business Start Date` / `Business End Date` / `Is Active Flag` | CHAR(18) — SCD columns wrong type |

#### 4.2.3 Metadata Completeness

Same gaps as `Credit_Card_Account`: `effective_from`, `effective_to`, `is_current` typed CHAR(18); `run_id` and `source_ingestion_date` missing.

---

#### Findings

### Finding CCT-001 — Missing Surrogate and Business Key

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | `SLV-001`, `SLV-002` |
| Evidence | No `credit_card_transaction_sk` or `credit_card_transaction_bk` in logical model or physical table |
| Affected Table | `silver_dev.credit_card.credit_card_transaction` |
| Affected Column(s) | `credit_card_transaction_sk` (missing), `credit_card_transaction_bk` (missing) |
| Confidence | 0.99 |

**Description:** Same structural absence as in the Account entity. `Transaction Id` from CTRANSACTIONS.SERNO is the candidate BK.

**Remediation:** Add `credit_card_transaction_sk STRING NOT NULL` (MD5), `credit_card_transaction_bk STRING NOT NULL`.

**Estimated Effort:** Medium
**Owner:** Data Modeling + ETL

---

### Finding CCT-002 — Duplicate/Ambiguous Source Mapping: Transaction Amount and Source Currency Amount

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | High |
| Guideline Rule | Mapping lineage integrity |
| Evidence | Mapping rows 149 and 172: `Transaction Amount` ← `CTRANSACTIONS.I004_AMT_TRXN`; `Source Currency Amount` ← `CTRANSACTIONS.I004_AMT_TRXN` (same column) |
| Affected Table | `silver_dev.credit_card.credit_card_transaction` |
| Affected Column(s) | `transaction_amount`, `source_currency_amount` |
| Confidence | 0.97 |

**Description:** `Transaction Amount` (billing currency) and `Source Currency Amount` (originating currency) both map to the same source column. These are semantically different: one is the settled amount in AED, the other is the amount in the card-holder's transaction currency. The mapping is incorrect — `source_currency_amount` should map to `I004_AMT_TRXN` only when `I049_CUR_TRXN ≠ 'AED'`, with a separate conversion column. Retaining the same value in both fields will understate FX exposure.

**Remediation:** Source `transaction_amount_aed` from settled amount (post-FX conversion); source `source_currency_amount` from `I004_AMT_TRXN` in originating currency; add `transaction_currency_code` from `I049_CUR_TRXN`. Type both amount fields as `DECIMAL(18,4)`.

**Estimated Effort:** Medium
**Owner:** ETL / Business Analyst

---

### Finding CCT-003 — PII: Card Number Masking Not Confirmed

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | Gate 2 standard — "Card Numbers masked via SHA-256"; PDPL Article 5 |
| Evidence | Mapping row 158: `Card Number` ← `CARDX.NUMBERX (CHAR(25))`; no SHA-256 transformation noted |
| Affected Table | `silver_dev.credit_card.credit_card_transaction` |
| Affected Column(s) | `card_number` |
| Confidence | 0.88 |

**Description:** The Card Number is a 16-digit PAN (Primary Account Number) — the most sensitive PCI-DSS data element. Gate 2 requires SHA-256 masking. No transformation logic is documented in the mapping for this column. Storing unmasked PANs in Silver would violate PCI-DSS scope and UAE PDPL.

**Remediation:** Apply `SHA2(card_number, 256)` in the ETL transformation. Mark `PII = Y` in the mapping document. Register in data catalog with `PCI_SENSITIVE` classification tag.

**Estimated Effort:** Small
**Owner:** ETL / Security

---

### Finding CCT-004 — Semantic Mismatch: Deposit Account Segment Code

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | Medium |
| Guideline Rule | Mapping correctness |
| Evidence | Mapping row 171: `Deposit Account Segment Code` ← `CTRANSACTIONS.MSGTYPE (NUMBER(10,0))` |
| Affected Table | `silver_dev.credit_card.credit_card_transaction` |
| Affected Column(s) | `deposit_account_segment_code` |
| Confidence | 0.90 |

**Description:** `MSGTYPE` is the ISO 8583 message type indicator (e.g., 0100 = authorization, 0200 = financial transaction). This is not a deposit account segment code. Mapping it to `deposit_account_segment_code` will produce meaningless or misleading values in this field.

**Remediation:** Identify the correct source for deposit segment if applicable to card transactions, or remove this attribute from the transaction entity (it belongs in a customer/account context). Retain `MSGTYPE` as `iso_message_type_code`.

**Estimated Effort:** Small
**Owner:** ETL / Business Analyst

---

### Finding CCT-005 — All Monetary and Date Attributes Typed STRING

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | `SLV-010`, Gate 2 — "DECIMAL(18,4) for monetary" |
| Evidence | Logical model: `Transaction Amount`, `Source Currency Amount` typed STRING; source types NUMBER(22), NUMBER(16,3) |
| Affected Table | `silver_dev.credit_card.credit_card_transaction` |
| Affected Column(s) | `transaction_amount`, `source_currency_amount`, `transaction_date`, `transaction_posted_date`, `transaction_time` |
| Confidence | 0.99 |

**Description:** Same structural issue as Account entity. Financial calculations on transaction amounts will be incorrect or impossible.

**Remediation:** Retype monetary → `DECIMAL(18,4)`, dates → `DATE`, times/timestamps → `TIMESTAMP`.

**Estimated Effort:** Large
**Owner:** Data Modeling + ETL

---

### Finding CCT-006 — Missing SCD-2 and Metadata Columns

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | `SLV-003`, `SLV-008`, guideline §2.7 |
| Evidence | Logical model: `Business Start Date CHAR(18)`, `Business End Date CHAR(18)`, `Is Active Flag CHAR(18)`; missing `run_id`, `source_ingestion_date` |
| Affected Table | `silver_dev.credit_card.credit_card_transaction` |
| Affected Column(s) | `effective_from` (missing), `effective_to` (missing), `is_current` (missing), `run_id` (missing) |
| Confidence | 0.99 |

**Description:** Identical SCD-2 metadata gap as the Account entity.

**Remediation:** Same as CCA-006 remediation applied to the transaction entity.

**Estimated Effort:** Medium
**Owner:** Data Modeling + ETL

---

### Finding CCT-007 — Missing Card Authorization Sub-Entity

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | High |
| Guideline Rule | SA taxonomy (`sa/subject-areas.md`): "Card authorization and audit log" explicitly listed |
| Evidence | No `credit_card_authorization` entity in logical model or physical tables |
| Affected Table | N/A — entity is missing |
| Affected Column(s) | N/A |
| Confidence | 0.95 |

**Description:** Authorization events (ISO 8583 0100/0110 messages) are pre-settlement events that are critical for fraud analysis, dispute management, and card limit enforcement. The SA taxonomy explicitly names "Card authorization audit log" as in scope. Currently, authorization data appears to be partially embedded in the transaction entity via `Network Affiliation Id`, `Online Transaction Indicator`, `Electronic Commerce Indicator` — this is the wrong grain.

**Remediation:** Design and create `credit_card_authorization` entity with grain = one row per authorization attempt.

**Estimated Effort:** Large
**Owner:** Data Modeling

---

### 4.3 Credit_Card_Statement

**State**: 🔴 Incorrect | **Criticality**: High | **Confidence**: 0.92
**Physical Table**: **MISSING** — no `credit_card_statement` table in `silver_dev.credit_card`
**Sources**: PRIME/TCTDBS (primary), FLEXUAT/FCUBSPROD

#### 4.3.1 Industry Fitness

The statement entity represents the monthly billing cycle summary — a critical entity for collections, customer servicing, and regulatory disclosures. Its absence as a physical table means the billing lifecycle is not implemented despite 74% mapping coverage in the data mapping document. The entity contains aggregate fields (`Total Purchase Amount`, `Total Cash Advance Amount`, `Purchase Cnt`, `Cash Advance Cnt`) which per `SLV-007` must not exist in Silver.

The logical model mixes the statement header (billing period, currency, account) with summary counts and totals, and also includes `Account type` — a denormalized FK attribute. The `New balance` attribute maps to `OPENINGBALANCE` in the mapping — this is a semantic inversion (opening balance ≠ new balance) and a data quality risk.

#### 4.3.2 Attribute-Level Review (Key Issues)

| Attribute | Type | Issue |
|---|---|---|
| `credit_card_statement_sk` | Missing | No surrogate key |
| `Statement Document ID` | STRING | Should be `credit_card_statement_bk` |
| `Total Purchase Amount` | STRING | **Aggregate — SLV-007 violation**; should be in Gold |
| `Total Cash Advance Amount` | STRING | **Aggregate — SLV-007 violation** |
| `Purchase Cnt` | STRING | **Count metric — SLV-007 violation** |
| `Cash Advance Cnt` | STRING | **Count metric — SLV-007 violation** |
| `New balance` | STRING | Maps to `OPENINGBALANCE` — **semantic inversion** |
| `Billing_period_end` | STRING | Maps to `CStatements.CYCLEDAYS (NUMBER(5))` — number of days, not end date |
| `Statement date` | STRING | Source is DATETIME; should be TIMESTAMP |
| `Account type` | STRING | Denormalized from Account — should be FK only |
| `Status` | STRING | Maps to `REASON (CHAR(1))` — semantic mismatch (REASON ≠ Status) |
| `Business Start Date` / `End` / `Is Active Flag` | CHAR(18) | SCD wrong type |

#### 4.3.3 Metadata Completeness

Same gaps as above. Physical table absent entirely.

---

#### Findings

### Finding CCS-001 — Physical Table Missing

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | Guideline §2.11 — "Tables that exist in the platform but have no corresponding entity in the Erwin model are a governance gap" (inverse also true) |
| Evidence | Physical structures CSV: no `credit_card_statement` table in `silver_dev.credit_card`; data mapping has 22 mapped attributes |
| Affected Table | `silver_dev.credit_card.credit_card_statement` (to be created) |
| Affected Column(s) | All |
| Confidence | 0.99 |

**Description:** The Statement entity has 22 mapped attributes in the data mapping document but no deployed physical table. The billing statement is a core revolving credit artifact required for customer communications, collections triggers, and CBUAE monthly reporting.

**Remediation:** Design DDL and deploy `silver_dev.credit_card.credit_card_statement` (post-fix of all type, SK, and SLV-007 issues below).

**Estimated Effort:** Medium
**Owner:** ETL / DBA

---

### Finding CCS-002 — Aggregate Metrics Violate SLV-007

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | High |
| Guideline Rule | `SLV-007` — "No pre-computed metrics"; `SLV-006` — "No derived values/aggregations in Silver" |
| Evidence | Logical model attrs: `Total Purchase Amount`, `Total Cash Advance Amount`, `Purchase Cnt`, `Cash Advance Cnt` |
| Affected Table | `silver_dev.credit_card.credit_card_statement` |
| Affected Column(s) | `total_purchase_amount`, `total_cash_advance_amount`, `purchase_cnt`, `cash_advance_cnt` |
| Confidence | 0.97 |

**Description:** Statement summary counts and totals are pre-computed aggregates of `Credit_Card_Transaction` rows. They should be computed in Gold from atomic transaction records, not stored in Silver.

**Remediation:** Remove these four columns from the Silver entity. Build `gold.credit_card.fact_statement_summary` that aggregates from `credit_card_transaction`.

**Estimated Effort:** Small (removal) + Medium (Gold build)
**Owner:** Data Modeling

---

### Finding CCS-003 — Semantic Inversion: New Balance Maps to Opening Balance

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | High |
| Guideline Rule | Mapping correctness / data quality |
| Evidence | Mapping row 211: `New_balance` ← `CStatements.OPENINGBALANCE` |
| Affected Table | `silver_dev.credit_card.credit_card_statement` |
| Affected Column(s) | `new_balance` |
| Confidence | 0.93 |

**Description:** The "new balance" (end-of-period balance) is mapped to the source column `OPENINGBALANCE`. These are opposite ends of the billing cycle. This mapping will make the new balance equal to the prior period's opening balance — a material error for collections and customer statements.

**Remediation:** Investigate correct source column for end-of-period balance (likely `CLOSINGBALANCE`). Verify mapping with PRIME system SME.

**Estimated Effort:** Small
**Owner:** ETL / Business Analyst

---

### Finding CCS-004 — Billing Period End Mapped to Days Count (Not End Date)

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | Medium |
| Guideline Rule | Mapping correctness; type accuracy |
| Evidence | Mapping row 204: `Billing_period_end` ← `CStatements.CYCLEDAYS (NUMBER(5))` — a cycle day count, not a date |
| Affected Table | `silver_dev.credit_card.credit_card_statement` |
| Affected Column(s) | `billing_period_end` |
| Confidence | 0.92 |

**Description:** The billing period end date is mapped to `CYCLEDAYS` — a numeric column representing the number of days in the cycle, not the actual end date. This will store a number (e.g., "30") in a date field.

**Remediation:** Derive billing period end as `billing_period_start + CYCLEDAYS - 1` or find the explicit end date column in PRIME.CSTATEMENTS. Type field as `DATE`.

**Estimated Effort:** Small
**Owner:** ETL

---

### 4.4 Credit_Card_Statement_Line_Item

**State**: 🔴 Incorrect | **Criticality**: High | **Confidence**: 0.88
**Physical Table**: `silver_dev.credit_card.credit_card_statement_line_item`
**Note**: The mapping sheet uses entity name `Card_Statement_Line_Item` (inconsistent with logical model `Credit_Card_Statement_Line_Item`)

#### 4.4.1 Industry Fitness

This entity represents individual line entries on a billing statement. It should have grain = one row per statement line item. The logical model has 26 attributes but the mapping only covers 9 (35% coverage — the lowest of mapped entities with physical tables). The entity duplicates several fields from the transaction entity (`Transaction Date`, `Transaction amount`, `Transaction currency code`, `Merchant name`) creating denormalization rather than a proper FK to `Credit_Card_Transaction`.

#### 4.4.2 Attribute-Level Review (Key Issues)

| Attribute | Issue |
|---|---|
| `credit_card_statement_line_item_sk` | Missing surrogate key |
| `Transaction Date` | Duplicate from transaction (denorm), STRING type |
| `Transaction amount` | Duplicate from transaction, STRING type (monetary) |
| `Transaction type` | Duplicate code from transaction, STRING |
| `Transaction description` | Duplicate from transaction |
| `Transaction currency code` | Duplicate from transaction |
| `Merchant name or identifier` | Duplicate from transaction |
| Statement parent FK | Should be `credit_card_statement_sk` NOT NULL FK |
| `Agreement statement line item amount` | Maps to `TBALANCES.OUTSTANDINGAMOUNT` — STRING, should be DECIMAL(18,4) |
| `Statement Organisation Business type code` | Not Available in primary source |

---

#### Findings

### Finding CCSI-001 — Extensive Denormalization: Transaction Attributes Copied to Statement Line

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | Medium |
| Guideline Rule | `SLV-006` — "3NF; no derived values" |
| Evidence | Logical model: `Transaction Date`, `Merchant name`, `Transaction amount`, `Transaction type`, `Transaction description`, `Transaction currency code` all duplicated from `Credit_Card_Transaction` |
| Affected Table | `silver_dev.credit_card.credit_card_statement_line_item` |
| Affected Column(s) | 6 transaction attributes |
| Confidence | 0.90 |

**Description:** Statement line item should only carry `Transaction Id` (FK to `credit_card_transaction`) plus statement-specific fields (line item number, amount on statement, statement document FK). Duplicating transaction attributes creates update anomalies and increases storage.

**Remediation:** Remove duplicated transaction attributes from this entity. Retain only `credit_card_transaction_bk` as FK. Transaction attributes accessed via Gold join.

**Estimated Effort:** Medium
**Owner:** Data Modeling

---

### Finding CCSI-002 — Missing Surrogate Key + 73% Attribute Gap

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | `SLV-001`, `SLV-002`; mapping completeness |
| Evidence | 9 of 26 logical attributes mapped (35%); no `_sk` or `_bk` |
| Affected Table | `silver_dev.credit_card.credit_card_statement_line_item` |
| Affected Column(s) | All unmapped columns, both key columns |
| Confidence | 0.92 |

**Description:** With 17 unmapped attributes and no identity key, this entity is functionally incomplete. Physical table exists but is not reliably usable.

**Remediation:** Complete the mapping for the 17 unmapped attributes; add `credit_card_statement_line_item_sk` and `credit_card_statement_line_item_bk`.

**Estimated Effort:** Large
**Owner:** ETL / Data Modeling

---

### 4.5 Credit_Card_Payment_Schedule

**State**: 🔴 Incorrect | **Criticality**: High | **Confidence**: 0.90
**Physical Table**: `silver_dev.credit_card.credit_card_payment_schedule`

#### 4.5.1 Industry Fitness

Payment schedule represents the instalment plan associated with a revolving credit account (e.g., EMI plans, balance conversion). The logical model has 21 attributes but only 4 are mapped (19% coverage). The entity is linked to Invoice through `Consolidated Invoice ID` and `Invoice ID`, but the Invoice entity has no physical table — FK integrity is broken.

#### 4.5.2 Attribute-Level Review (Key Issues)

| Attribute | Issue |
|---|---|
| `credit_card_payment_schedule_sk` | Missing |
| `Payment Scheduled amount Date Time` | STRING; should be TIMESTAMP |
| `Payment Scheduled amount local amount` | STRING; source NUMBER(16,3); should be DECIMAL(18,4) |
| `Payment Scheduled amount transaction amount` | STRING; source NUMBER(16,3); should be DECIMAL(18,4) |
| `Payment Scheduled amount Global amount` | STRING; source NUMBER(16,3); should be DECIMAL(18,4) |
| FK to `Credit_Card_Invoice` | Broken — Invoice table does not exist |
| 17 of 21 attributes | Unmapped (81% gap) |

---

#### Findings

### Finding CCPS-001 — 81% Unmapped Attributes

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | High |
| Guideline Rule | Mapping completeness; SLV-001 |
| Evidence | Entity Summary shows 4 mapped attributes of 21 logical attributes |
| Affected Table | `silver_dev.credit_card.credit_card_payment_schedule` |
| Affected Column(s) | All 17 unmapped attributes |
| Confidence | 0.95 |

**Description:** `Payment schedule Term code`, `Payment Scheduled amount due Date Time`, `Account number`, `Account type` and all other schedule metadata (term, grace period, frequency) are unmapped. The schedule entity is unusable for collections or customer service use cases.

**Remediation:** Complete source mapping for remaining 17 attributes; reference `Reference_Credit_Card_Payment_Schedule_Term` for FK term code.

**Estimated Effort:** Large
**Owner:** ETL

---

### Finding CCPS-002 — Broken FK to Credit_Card_Invoice

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | `SLV-005` referential integrity; mapping §3 Gate 3 |
| Evidence | Physical: no `credit_card_invoice` table in `silver_dev.credit_card`; logical FK via `Consolidated Invoice ID` and `Invoice ID` |
| Affected Table | `silver_dev.credit_card.credit_card_payment_schedule` |
| Affected Column(s) | `consolidated_invoice_id`, `invoice_id` |
| Confidence | 0.99 |

**Description:** The payment schedule references an invoice entity that has no physical table. Any pipeline writing payment schedule records with invoice FKs will produce dangling references.

**Remediation:** Deploy `credit_card_invoice` physical table (per Finding CCI-001) before populating payment schedule records that reference invoices.

**Estimated Effort:** Large (dependent on Invoice entity remediation)
**Owner:** Data Modeling + ETL

---

### 4.6 Credit_Card_Receivables

**State**: 🟡 Partial | **Criticality**: Medium | **Confidence**: 0.88
**Physical Table**: `silver_dev.credit_card.credit_card_receivables`
**Source**: Intellect/COLLECTION (primary)

#### 4.6.1 Industry Fitness

Receivables/collections activity represents the recovery workflow for delinquent credit card accounts. This entity belongs architecturally to `silver.collection` (Lending/Collections SA) per the subject area taxonomy, which states: "Collections lifecycle starts and ends within the lending credit relationship." Including revolving credit collection activity in the Card SA creates a cross-SA scope overlap. However, given the current separation of Credit Card from Lending in the physical implementation, this assessment treats it as in-scope for the Card SA.

#### 4.6.2 Attribute-Level Review (Key Issues)

| Attribute | Issue |
|---|---|
| `credit_card_receivables_sk` | Missing |
| `Receivables collect start Date` ↔ `Receivables collect End Date` | Both map to `COLLTRANSPAY.CP_PAY_DATE` — same source column; end date is incorrect |
| `Dunning Level Cd` | Not Available |
| `Account number` / `Account type` | Present in logical model but absent from mapping |
| All dates | STRING type; source DATE |

---

#### Findings

### Finding CCR-001 — Duplicate Source Mapping: Collection Start and End Date

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | High |
| Guideline Rule | Mapping lineage integrity |
| Evidence | Mapping rows 118–119: both `Receivables collect start Date` and `Receivables collect End Date` ← `COLLTRANSPAY.CP_PAY_DATE` |
| Affected Table | `silver_dev.credit_card.credit_card_receivables` |
| Affected Column(s) | `receivables_collect_end_date` |
| Confidence | 0.94 |

**Description:** Start and end of collection activity map to the same source column. End date will always equal start date, making duration calculations meaningless.

**Remediation:** Identify the correct source column for collection end date (e.g., `CP_END_DATE` or derived from next action event). Update mapping.

**Estimated Effort:** Small
**Owner:** ETL / Business Analyst

---

### Finding CCR-002 — Scope Overlap with Collections SA

| Field | Value |
|---|---|
| Priority | P2 |
| Criticality | Medium |
| Guideline Rule | `sa/subject-areas.md` — "Collections is under Lending" |
| Evidence | SA taxonomy: `silver.collection` covers "Pre-Delinquent Monitoring, Delinquency Cases, Collection Actions" |
| Affected Table | `silver_dev.credit_card.credit_card_receivables` |
| Affected Column(s) | Entire entity |
| Confidence | 0.85 |

**Description:** Revolving credit collection activity is scoped inside the Card SA but architecturally belongs in `silver.collection`. This creates duplication risk if a Lending/Collections SA is ever implemented.

**Remediation:** Plan migration of `credit_card_receivables` to `silver.collection` as a `collections_case` child entity with `facility_type_code = 'CREDIT_CARD'`.

**Estimated Effort:** Large
**Owner:** Data Modeling / Architecture

---

### 4.7 Credit_Card_Invoice

**State**: 🔴 Incorrect | **Criticality**: High | **Confidence**: 0.98
**Physical Table**: **MISSING**
**Mapping Coverage**: 0% (11 attributes, all Not Available)

#### 4.7.1 Industry Fitness

The invoice entity represents the periodic billing document issued to credit card holders — the formal trigger for payment obligation. Per BIAN *Credit Card* domain, the invoice/statement cycle is a core operation. With 0% mapping and no physical table, the entire billing lifecycle is absent from the Silver layer. This directly impacts customer service, collections triggers, and CBUAE card reporting.

#### 4.7.2 Attribute-Level Review

All 22 logical attributes are unmapped. Key attributes that are critical:

| Attribute | Criticality |
|---|---|
| `Invoice ID` | PK candidate — unmapped |
| `Account number` | FK to account — unmapped |
| `Invoice Date Time` | Core timestamp — unmapped |
| `Invoice currency code` | ISO 4217 — unmapped |
| `Invoice Type Code` | Classification — unmapped |

---

#### Findings

### Finding CCI-001 — Entity Entirely Absent: No Physical Table, 0% Mapping

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | `SLV-001`, `SLV-002`; SA completeness |
| Evidence | Physical CSV: no `credit_card_invoice` table; mapping: 11 attributes all `Not Available` |
| Affected Table | `silver_dev.credit_card.credit_card_invoice` (to be created) |
| Affected Column(s) | All |
| Confidence | 0.99 |

**Description:** The invoice entity is entirely unimplemented. No source mapping has been identified for any of the 11 logical attributes (plus 11 additional from the logical model not in the mapping). The billing cycle — a fundamental credit card operation — has no Silver representation. `Credit_Card_Payment_Schedule` references this entity via `Invoice ID` and `Consolidated Invoice ID`, making those FKs immediately broken.

**Remediation:** (1) Identify source system for credit card invoices — candidate is PRIME.TCTDBS.CTRANSACTIONS for invoice-type transactions. (2) Design DDL. (3) Map attributes. (4) Deploy ETL pipeline. (5) Validate FK from Payment Schedule.

**Estimated Effort:** Large
**Owner:** Data Modeling + ETL

---

### 4.8 Credit_Card_event

**State**: 🟡 Partial | **Criticality**: Medium | **Confidence**: 0.82
**Physical Table**: `silver_dev.credit_card.credit_card_event`
**Note**: This entity exists in the mapping and physical layer but is **not in the logical model**.

#### 4.8.1 Industry Fitness

The entity captures card-level reward events (reward flag, reward type, reward points). It is not present in the formal logical model but exists in the data mapping and physical schema. Per governance rules, all Silver tables must have a corresponding approved logical entity.

---

#### Findings

### Finding CCE-001 — Entity Not in Logical Model (Governance Gap)

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | Medium |
| Guideline Rule | Guideline §2.11 — "Tables that exist in the platform but have no corresponding entity in the Erwin model are a governance gap" |
| Evidence | Physical: `credit_card_event` exists; logical model: no `Credit_Card_event` entity in `CreditCard` SA |
| Affected Table | `silver_dev.credit_card.credit_card_event` |
| Affected Column(s) | All |
| Confidence | 0.88 |

**Description:** A physical Silver table exists without a corresponding approved logical model entity. This bypasses the "Erwin model before DDL" governance gate.

**Remediation:** Either (1) add `Credit_Card_event` to the logical model with full attribute documentation, or (2) migrate reward events to `silver.event.business_event` with `event_type_code = 'CARD_REWARD'`.

**Estimated Effort:** Small
**Owner:** Data Modeling / Governance

---

### Finding CCE-002 — Temp/Test Tables in Production Schema

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | Silver layer purpose — "single authoritative curated store"; no development artifacts in production |
| Evidence | Physical CSV: tables `credit_card_account_temp`, `credit_card_account_test2703`, `credit_card_event11` in `silver_dev.credit_card` |
| Affected Table | `silver_dev.credit_card.credit_card_account_temp`, `credit_card_account_test2703`, `credit_card_event11` |
| Affected Column(s) | Entire tables |
| Confidence | 0.99 |

**Description:** Three development/test tables exist in the Silver schema. These contaminate the authoritative data layer, may consume significant storage (the account table is 917MB — the `_temp` variant likely mirrors this), and create confusion in data catalog and lineage tools.

**Remediation:** Drop all three tables immediately: `DROP TABLE silver_dev.credit_card.credit_card_account_temp`, `DROP TABLE silver_dev.credit_card.credit_card_account_test2703`, `DROP TABLE silver_dev.credit_card.credit_card_event11`. Establish naming conventions policy to prevent development tables from entering production schemas.

**Estimated Effort:** Small
**Owner:** DBA

---

### 4.9 Loan_Instalment_Schedule

**State**: 🔴 Incorrect | **Criticality**: High | **Confidence**: 0.97
**Physical Table**: None found in `silver_dev.credit_card`

#### Findings

### Finding LIS-001 — Entity Misplaced in Credit Card SA

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | High |
| Guideline Rule | `sa/subject-areas.md` — "Lending is the top-level subject area for all credit activity"; SA isolation |
| Evidence | Logical model: `Loan_Instalment_Schedule` in `CreditCard` SA; `sa/subject-areas.md` entry 8a Loan: "Credit facilities: Loans, Islamic structures, Repayment Schedules, Disbursements" |
| Affected Table | N/A |
| Affected Column(s) | Entire entity |
| Confidence | 0.97 |

**Description:** Loan instalment schedules belong to the Lending SA (`silver.lending`), not the Credit Card SA. Their inclusion here creates cross-SA scope bleed. If a Lending SA is ever built, this entity would be duplicated.

**Remediation:** Remove `Loan_Instalment_Schedule` from the Credit Card logical model. Ensure the Lending SA logical model covers this entity.

**Estimated Effort:** Small
**Owner:** Data Modeling / Architecture

---

### 4.10 Reference Entities (Consolidated)

**State**: 🟡 Partial | **Criticality**: Low–Medium | **Confidence**: 0.85
**Physical Tables**: 22 `reference_*` tables in `silver_dev.credit_card`

#### 4.10.1 Industry Fitness

The Credit Card SA contains 43 reference entities. Of these:
- **Correctly scoped** (12): `reference_agreement_status`, `reference_block_status`, `reference_monitor_status`, `reference_mcc`, `reference_product`, `reference_segment`, `reference_service_program`, `reference_transaction_code`, `reference_credit_card_payment_*`, `reference_cust_cards`
- **Should be in `silver.reference`** (6): `reference_exchange_rate`, `reference_currency_code`, `reference_account_currency`, `reference_interest_rate`, `reference_source_system_code`
- **Should be in `silver.org`** (1): `reference_bank_branch`
- **Should be in `silver.lending` or `silver.risk`** (2): `reference_loan_status`, `reference_esg_status`
- **Duplicates/orphans** (2): `reference_card` / `reference_cust_cards` — appear duplicated; `Reference_Account_X` is a hybrid entity mixing account type and customer attributes
- **Missing physical table** (7): Multiple reference entities in the logical model have no corresponding physical table

**All reference entities share the same type issues**: `CHAR(18)` SCD columns, STRING for all attributes, no `_sk`/`_bk` keys.

---

#### Findings

### Finding REF-001 — Cross-SA Reference Entities Violate SA Isolation

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | Medium |
| Guideline Rule | `SLV-006` — "Subject area isolation"; guideline §2.2 |
| Evidence | `reference_exchange_rate`, `reference_currency_code`, `reference_interest_rate` in `silver_dev.credit_card` |
| Affected Table | Multiple `reference_*` tables |
| Affected Column(s) | Entire entities |
| Confidence | 0.92 |

**Description:** Exchange rates, currency codes, and bank branches are global reference data owned by `silver.reference` and `silver.org` respectively. Duplicating them in the Credit Card schema creates: (1) maintenance overhead when rates change; (2) inconsistency across SAs if different values are loaded; (3) pipeline coupling that violates SA isolation.

**Remediation:** Remove these entities from `silver_dev.credit_card`. Credit Card pipelines should JOIN to `silver.reference.*` tables for these lookups.

**Estimated Effort:** Medium
**Owner:** Data Modeling + ETL

---

### Finding REF-002 — Reference_Group_Account_Type Fully Unmapped

| Field | Value |
|---|---|
| Priority | P2 |
| Criticality | Low |
| Guideline Rule | Mapping completeness |
| Evidence | Mapping: all 5 attributes `Not Available`; no physical table found |
| Affected Table | N/A |
| Affected Column(s) | All |
| Confidence | 0.90 |

**Description:** Group account type codes (used for grouped card accounts) have no source mapping identified.

**Remediation:** If group account functionality is not in scope for current phase, mark entity as `DEFERRED` in the logical model. Otherwise, identify source system and map.

**Estimated Effort:** Small
**Owner:** Data Modeling

---

## 5. Denormalization Register

| Entity | Denormalized Attribute | Classification | Justification Required | Finding |
|---|---|---|---|---|
| `Credit_Card_Account` | `Amount in arrears 1–8 period` (×8 buckets) | ❌ Unnecessary — 3NF violation | None documented | CCA-003 |
| `Credit_Card_Account` | `Fee table 1`, `Fee table 2` | ❌ Unnecessary | None documented | CCA-009 |
| `Credit_Card_Account` | `Customer number (CIF)` embedded | ⚠️ Acceptable (FK reference) | Standard FK — but should use `customer_bk` SK reference | CCA-001 |
| `Credit_Card_Account` | 3 separate credit limit columns with same source | ❌ Unnecessary — deduplication required | None documented | CCA-005 |
| `Credit_Card_Statement` | `Account type` embedded | ❌ Unnecessary | Should be FK to account only | CCS-001 (related) |
| `Credit_Card_Statement_Line_Item` | `Transaction Date`, `Merchant name`, `Transaction amount`, `Transaction type`, `Description`, `Currency code` | ❌ Unnecessary | Should be FK to transaction entity | CCSI-001 |
| `Credit_Card_Transaction` | `Short Name` (customer name) | ❌ Unnecessary | Party attribute embedded in transaction; denorm from `silver.party` | CCT-002 (related) |
| `Reference_*` (exchange rate, currency, branch) | Cross-SA data embedded in CC schema | ❌ Unnecessary | No justification; violates SA isolation | REF-001 |

**Notes**: The normalization defects are extensive. No denormalization in this subject area carries documented steward-approved justification. All ❌ items require remediation.

---

## 6. Guideline Compliance Summary

| Rule | Rule Summary | Status | Finding Reference |
|---|---|---|---|
| **SLV-001** | `<entity>_sk` (MD5 surrogate) on every entity | 🔴 FAIL | CCA-001, CCT-001, CCSI-002, CCI-001, CCPS-001 |
| **SLV-002** | `<entity>_bk` (business key) on every entity | 🔴 FAIL | All core entities |
| **SLV-003** | SCD-2 default; `effective_from`/`to`/`is_current` | 🔴 FAIL | CCA-006, CCT-006; wrong types on all entities |
| **SLV-004** | No source-system codes; use `code_mapping` | 🔴 FAIL | CCA-007; status codes stored raw |
| **SLV-005** | DQ-gated writes; quarantine table | 🟡 Unknown | No quarantine table found in physical structures |
| **SLV-006** | 3NF; no derived values/aggregations | 🔴 FAIL | CCA-003, CCA-009, CCS-002, CCSI-001 |
| **SLV-007** | No pre-computed metrics | 🔴 FAIL | CCS-002; aggregate counts/amounts in Statement |
| **SLV-008** | All metadata columns present with correct types | 🔴 FAIL | All entities; CHAR(18) instead of TIMESTAMP/BOOLEAN |
| **SLV-009** | All timestamps UTC | 🟡 Unknown | No timezone documentation in mapping |
| **SLV-010** | Monetary amounts in DECIMAL(18,4) / fils | 🔴 FAIL | CCA-002, CCT-005; all monetary fields typed STRING |
| **NAM-001** | snake_case, lowercase | 🟡 Partial | Physical tables use snake_case; logical model uses mixed case |
| **NAM-003** | Table naming: `<entity>` for Silver | 🟡 Partial | Tables prefixed with `credit_card_` which is redundant given the schema name `credit_card`; reference tables do not follow entity naming |
| **NAM-005** | No SQL reserved words | 🟢 PASS | No reserved words detected |
| **Gate 2 — PII Masking** | Card Numbers, EID masked via SHA-256 | 🔴 FAIL | CCT-003; no masking documented for Card Number |
| **Gate 2 — Monetary Type** | DECIMAL(18,4) for all monetary | 🔴 FAIL | All entities |
| **Gate 2 — Date Format** | `YYYY-MM-DD` / TIMESTAMP for all dates | 🔴 FAIL | All entities; dates stored as STRING |
| **§2.1 3NF** | No repeating groups | 🔴 FAIL | CCA-003; arrears buckets 1–8 |
| **§2.2 SA Isolation** | No cross-SA joins in Silver pipelines | 🔴 FAIL | REF-001; cross-SA entities embedded |
| **§2.4 Deterministic SK** | MD5(UPPER(TRIM(bk))) | 🔴 N/A | No SKs exist to evaluate |
| **§2.5 Code Canonicalization** | Canonical codes via `code_mapping` | 🔴 FAIL | CCA-007; raw CBS codes stored |
| **§2.6 No Aggregations** | Atomic facts only | 🔴 FAIL | CCS-002 |
| **§2.7 Audit Columns** | All 11 mandatory audit columns | 🔴 FAIL | All entities; 3–4 columns wrong type, 2 missing |
| **§2.9 UTC Timestamps** | All timestamps in UTC | 🟡 Unknown | Not verifiable from current artifacts |
| **§2.11 Erwin Before DDL** | Logical model before deployment | 🔴 FAIL | `credit_card_event` deployed without logical model entity |
| **§2.12 Catalog Registration** | Registered in CDGC with steward | 🟡 Unknown | Not verifiable from current artifacts |

**Overall Compliance Score**: 1/25 rules fully PASS; 17 FAIL; 7 Unknown/Partial.

---

## 7. Remediation Plan

### Sprint 1 (Immediate — P0 Blockers)

| # | Action | Rule | Effort | Owner |
|---|---|---|---|---|
| R1 | Add `credit_card_account_sk` (MD5), `credit_card_account_bk` NOT NULL to Account entity and table | SLV-001/002 | L | Data Modeling + ETL |
| R2 | Add `credit_card_transaction_sk`, `credit_card_transaction_bk` to Transaction entity | SLV-001/002 | M | Data Modeling + ETL |
| R3 | Add SK/BK to Statement, Statement Line Item, Payment Schedule, Receivables, Invoice | SLV-001/002 | L | Data Modeling + ETL |
| R4 | Retype all monetary fields to `DECIMAL(18,4)`; all dates to `DATE`; all timestamps to `TIMESTAMP`; boolean flags to `BOOLEAN` | SLV-010, Gate 2 | L | Data Modeling + ETL |
| R5 | Fix SCD-2 metadata: rename `Business Start/End Date` → `effective_from/to TIMESTAMP NOT NULL`; `Is Active Flag` → `is_current BOOLEAN` | SLV-003, §2.7 | M | Data Modeling |
| R6 | Add missing audit columns: `run_id STRING`, `source_ingestion_date TIMESTAMP` to all entities | SLV-008, §2.7 | M | ETL |
| R7 | Drop `credit_card_account_temp`, `credit_card_account_test2703`, `credit_card_event11` | Governance | S | DBA |
| R8 | Apply SHA-256 masking to `Card Number` in Transaction ETL; mark PII=Y | Gate 2 / PCI-DSS | S | ETL / Security |

### Sprint 2 (High — P0/P1 Critical Gaps)

| # | Action | Rule | Effort | Owner |
|---|---|---|---|---|
| R9 | Design and deploy `credit_card_invoice` physical table; complete source mapping for all 22 attributes | SA completeness | L | Data Modeling + ETL |
| R10 | Deploy `credit_card_statement` physical table (with SLV-007 fix: remove aggregate columns) | SLV-007, completeness | M | Data Modeling + ETL |
| R11 | Normalize arrears buckets 1–8 into `credit_card_arrears_bucket` child entity | SLV-006 §2.1 | M | Data Modeling |
| R12 | Remove duplicate/ambiguous mappings: Interest Rate / Cash Advance Rate; Transaction Amount / Source Currency Amount; Permanent Credit Limit duplicates | Lineage | S | ETL + BA |
| R13 | Canonicalize all status codes via `silver.reference.code_mapping` (Agreement Status, Account Status, Block Status, Monitor Status) | SLV-004 | M | ETL |
| R14 | Fix `New_balance` → `CLOSINGBALANCE` mapping; fix `Billing_period_end` → derived end date | Accuracy | S | ETL + BA |
| R15 | Remove cross-SA reference entities from credit card schema: exchange rate, currency code, bank branch, loan status, ESG status | §2.2 SA Isolation | M | Data Modeling + ETL |
| R16 | Add `Credit_Card_event` to logical model or migrate reward events to `silver.event` | §2.11 | S | Data Modeling |

### Sprint 3 (P1 — Design Improvements)

| # | Action | Rule | Effort | Owner |
|---|---|---|---|---|
| R17 | Remove denormalized transaction attributes from Statement Line Item; add `credit_card_transaction_bk` FK | SLV-006 | M | Data Modeling |
| R18 | Design `card_fee_schedule` child entity for fee table normalization | SLV-006 §2.1 | M | Data Modeling |
| R19 | Complete mapping for Statement Line Item (17 unmapped attrs), Payment Schedule (17 unmapped), Receivables (12 unmapped) | Completeness | L | ETL |
| R20 | Design and deploy `credit_card_authorization` entity for ISO 8583 auth events | SA taxonomy | L | Data Modeling + ETL |
| R21 | Move `Loan_Instalment_Schedule` to Lending SA logical model | SA scoping | S | Data Modeling |
| R22 | Fix collection start/end date mapping in Receivables | Accuracy | S | ETL |

### Sprint 4 (P2 — Backlog)

| # | Action | Rule | Effort | Owner |
|---|---|---|---|---|
| R23 | Align SA canonical name to either "Card" (per taxonomy) or "Credit Card" (per implementation); update catalog | NAM, Governance | S | Governance |
| R24 | Rename SA directory from `sa/credit-card/` to `sa/credit_card/` for consistency | Governance | S | DevOps |
| R25 | Plan migration of `credit_card_receivables` to `silver.collection` | SA scoping | L | Architecture |
| R26 | Register all Credit Card Silver tables in Informatica GDGC with steward, classification, lineage | §2.12 | M | Data Governance |
| R27 | Mark `Reference_Group_Account_Type` as DEFERRED or source and map | Completeness | S | Data Modeling |
| R28 | Validate UTC timestamp compliance across all pipeline code | SLV-009 | M | ETL |

---

## 8. Appendix

### A — Logical to Physical Mapping Summary

| Logical Entity | Physical Table | Mapping Source | Mapped % | Notes |
|---|---|---|---|---|
| `Credit_Card_Account` | `credit_card_account` | PRIME/TCTDBS, CAPS/CAPSADMIN, FLEXUAT/FCUBSPROD | ~75% | 3 sources; 15 Not Available |
| `Credit_Card_Transaction` | `credit_card_transaction` | PRIME/TCTDBS, FLEXUAT/FCUBSPROD | ~96% | 2 Not Available; duplicate mappings |
| `Credit_Card_Statement` | **MISSING** | PRIME/TCTDBS, FLEXUAT/FCUBSPROD | ~71% | Physical table not deployed |
| `Credit_Card_Statement_Line_Item` | `credit_card_statement_line_item` | PRIME/TCTDBS | ~35% | 17 unmapped attrs |
| `Credit_Card_Payment_Schedule` | `credit_card_payment_schedule` | PRIME/TCTDBS, FLEXUAT/FCUBSPROD | ~19% | 17 unmapped attrs |
| `Credit_Card_Receivables` | `credit_card_receivables` | Intellect/COLLECTION | ~40% | 12 unmapped attrs |
| `Credit_Card_Invoice` | **MISSING** | None | **0%** | Entirely unimplemented |
| `Credit_Card_event` *(mapping only)* | `credit_card_event` | PRIME/TCTDBS, FLEXUAT/FCUBSPROD | N/A | Not in logical model |
| `Loan_Instalment_Schedule` | None | N/A | 0% | Belongs in Lending SA |
| Reference entities (39) | Various `reference_*` (22 physical) | PRIME/TCTDBS | Variable | 17 without physical tables |

**Source Systems Identified**:
- **PRIME / TCTDBS** — Core credit card processing system (Oracle); primary source for accounts, transactions, statements
- **CAPS / CAPSADMIN** — Card Authorization and Processing System; primary for card status, limits, delinquency
- **FLEXUAT / FCUBSPROD** — Flexcube UAT/Production; secondary source for account and customer data
- **Intellect / COLLECTION** — Collection management system; source for receivables collection activity

### B — Guideline Citations (Full Rule Text)

| Rule Code | Source Document | Full Rule Text |
|---|---|---|
| SLV-001 | `guidelines/11-modeling-dos-and-donts.md §2.4` | "Derive surrogate keys deterministically: `MD5(UPPER(TRIM(business_key)))` for single-component keys. Use `<entity>_sk` naming." |
| SLV-002 | `guidelines/11-modeling-dos-and-donts.md §2.4` | "Every Silver entity must have an `<entity>_bk` (business/natural key) column. Source-system IDs (`<entity>_id`) are Bronze-only." |
| SLV-003 | `guidelines/11-modeling-dos-and-donts.md §2.3` | "Track all changes to tracked attributes with SCD Type 2: add a new row with updated values, close the prior row. Every Silver entity table must have `effective_from`, `effective_to`, `is_current`." |
| SLV-004 | `guidelines/11-modeling-dos-and-donts.md §2.5` | "Resolve all source-system codes to canonical platform values using a central code-mapping reference (`silver.reference.code_mapping`). Store the canonical code, not the raw source code, in entity columns." |
| SLV-005 | `guidelines/01-data-quality-controls.md §3.2` | "Failed records routed to quarantine / error tables rather than discarded. All DQ results must be logged and auditable." |
| SLV-006 | `guidelines/11-modeling-dos-and-donts.md §2.1` | "Decompose entities into 3NF: every non-key attribute depends on the whole primary key and nothing but the primary key. Repeating groups must become child tables." |
| SLV-007 | `guidelines/11-modeling-dos-and-donts.md §2.6` | "Store only atomic, un-aggregated facts in Silver. DON'T pre-compute totals, averages, counts, ratios, or any derived metrics in Silver tables." |
| SLV-008 | `guidelines/11-modeling-dos-and-donts.md §2.7` | "Include the following Silver technical audit columns on every Silver entity table: `source_system_code`, `source_system_id`, `create_date`, `update_date`, `delete_date`, `is_active_flag`, `effective_from`, `effective_to`, `is_current`, `run_id`, `source_ingestion_date`." |
| SLV-009 | `guidelines/11-modeling-dos-and-donts.md §2.9` | "Store all timestamps in UTC. Convert source-local timestamps at ingestion. Where the source timezone is significant, store a companion `_source_tz` column." |
| SLV-010 | `guidelines/01-data-quality-controls.md §2 Gate 2` | "All monetary columns must use `DECIMAL(18,4)`." |
| NAM-001 | `guidelines/06-naming-conventions.md` | "Use snake_case for all identifiers. Use lowercase only. No abbreviations unless universally understood." |
| NAM-003 | `guidelines/06-naming-conventions.md` | "Silver entity table naming: `<entity>` (e.g., `customer`, `account`, `transaction`)." |
| NAM-005 | `guidelines/06-naming-conventions.md` | "Never use SQL reserved words as identifiers (e.g., `date`, `value`, `order`)." |
| Gate 2 PII | `guidelines/01-data-quality-controls.md §2 Gate 2` | "EID, Card Numbers, Phone Numbers, and Passport Numbers masked via SHA-256 unless stored in a restricted container." |

**Note on Guideline Conflict**: `guidelines/01-data-quality-controls.md §5` defines audit column names as `record_source`, `valid_from_at`, `valid_to_at`, `ingested_at`, `processed_at`, `batch_id`, `data_quality_status`, `dq_issues`. This conflicts with `guidelines/11-modeling-dos-and-donts.md §2.7` which defines `source_system_code`, `effective_from`, `effective_to`, `create_date`, `run_id`, `source_ingestion_date`. **Recommendation**: The Group Data Office should publish a single authoritative list of Silver audit columns resolving this conflict. This assessment uses the §2.7 list (from the modeling guide) as it is more prescriptive.

### C — Industry Standard References

| Standard | Relevance to Credit Card SA |
|---|---|
| **BIAN v11 — Credit Card** (§5.4) | Defines Card Agreement, Card Transaction, Card Authorization, Card Statement service operations. All core entities map to BIAN service domains. |
| **IFW (IBM/Teradata)** — `FINANCIAL_ACCOUNT`, `CARD_AGREEMENT` | `Credit_Card_Account` maps to IFW `CARD_AGREEMENT`; `Credit_Card_Transaction` maps to `FINANCIAL_TRANSACTION`. IFW separately models `ACCOUNT_BALANCE` from `CARD_AGREEMENT` — the current model conflates these. |
| **IFRS 9** | Revolving credit (credit cards) requires ECL staging (Stage 1/2/3) based on days-past-due. `delinquent_code` and `regulators_classification_code` are critical CDEs for IFRS 9 compliance. Currently unmapped (CCA-008). |
| **BCBS 239** | Principle 2 (Data Architecture): Credit card data must support risk data aggregation. Absence of SK keys and STRING typing on monetary fields directly violates BCBS 239 accuracy and integrity requirements. |
| **PCI-DSS v4.0** | Requirement 3: PANs (Card Numbers) must be rendered unreadable in storage. SHA-256 hashing is an acceptable rendering method. Finding CCT-003 directly references this requirement. |
| **CBUAE Circular 2023/7** | UAE Credit Card Regulatory Reporting: Monthly balance, delinquency classification, and payment schedule data required. `Regulators classification code` (unmapped) feeds this report. |
| **UAE PDPL** | Article 5: Processing of personal data must be lawful, fair, and transparent. Card numbers and CIF numbers are personal data requiring consent and masking controls. |

---

*Report generated: 2026-04-27T14:23:26Z*
*Assessment version: 1.0*
*Assessor: Automated Silver-Layer Gap Assessment (LLM Senior Data Architect)*
*Next review due: Post-Sprint 2 completion*
