
---

# Account Subject Area — Silver Layer Gap Assessment Report

**Assessment Date:** 2026-04-19  
**Assessor:** Automated Gap Assessment (LLM-assisted)  
**Repository:** imgokhan-rakbank/logical-model-assessment  
**Target Schema:** `silver.account` (catalogued as `silver_dev_v2.account`)  
**Guideline Version:** Data Platform Development Standards V1.0 Draft  
**Confidence (Overall):** 0.92

---

## 1. Executive Summary

### 1.1 Management Slide

| Dimension | Score | Status |
|---|---|---|
| **SA Architecture & Scoping** | 2 / 10 | 🔴 Critical |
| **Entity Coverage (logical→physical)** | 2 / 10 | 🔴 Critical |
| **Attribute Quality & Data Types** | 2 / 10 | 🔴 Critical |
| **Key Strategy (SK/BK)** | 1 / 10 | 🔴 Critical |
| **SCD-2 / History** | 3 / 10 | 🔴 Critical |
| **Mandatory Metadata Completeness** | 3 / 10 | 🔴 Critical |
| **Naming Convention Compliance** | 4 / 10 | 🟡 High |
| **DQ Controls & Lineage** | 3 / 10 | 🔴 Critical |
| **3NF / Denormalization** | 3 / 10 | 🔴 Critical |
| **Overall** | **2.6 / 10** | 🔴 Critical |

**Summary statement:** The Account subject area (`silver.account`) is in a pre-production, exploratory state. Only 2 of 30 logical entities have a corresponding physical table. The single most-populated physical table (`account_agreement_master`) exists but conflates account master data with balance metrics, joint-holder identity, product descriptions, and KYC status — all of which violate 3NF. Fifteen physical tables exist with no corresponding logical entity, and several of those tables belong to other subject areas entirely (currency/FX → `silver.reference`; financial instruments → `silver.treasury`). The core canonical `account` entity with `account_sk`/`account_bk` is absent. Surrogate keys are absent from all logical entities. Every monetary attribute is mapped as STRING, violating the monetary-amount convention. Eight physical tables are empty (0 bytes). Immediate P0 action is required before any Gold or reporting layer can consume this subject area.

### 1.2 Top 5 Priority Actions

| # | Action | Priority | Effort |
|---|---|---|---|
| 1 | Create canonical `account` entity (grain: one row per account per SCD-2 version) with `account_sk` (MD5 surrogate), `account_bk`, all mandatory SCD-2 and audit metadata columns | **P0** | Large |
| 2 | Relocate out-of-scope entities: collateral/exposure → `silver.collateral`; currency/FX → `silver.reference`; financial instruments → `silver.treasury`; `Agreement` → `silver.contract`; risk/grade entities → `silver.risk`; access device → `silver.event` | **P0** | Large |
| 3 | Fix all attribute data types: monetary fields → `DECIMAL(18,4)` (Silver) with AED amounts in smallest unit (fils, `BIGINT`); date fields → `DATE`; timestamp fields → `TIMESTAMP` (UTC); eliminate STRING-for-everything | **P0** | Large |
| 4 | Add `account_sk` (MD5 deterministic) + `account_bk` surrogate/business key pair to every entity; remove all source-system natural IDs from PK position | **P0** | Medium |
| 5 | Decompose `Account_Agreement_Master`: extract balance metrics to balance entity, joint holders to party-role child table, KYC status to Party SA, product description to Product SA reference | **P1** | Large |

---

## 2. Subject Area Architecture Assessment

### 2.1 Scoping vs. `sa/subject-areas.md`

**Canonical definition (subject-areas.md #5):**
> *Account — `silver.account` — "Holding Containers: Unified Balance records for CASA, Term Deposits, Loan Accounts, and Card Balances. Tracks lifecycle status." Priority: 1 (Critical)*

**Actual physical scope (18 tables found):**

| Physical Table | Assessed Scope | Correct SA? |
|---|---|---|
| `account_agreement_master` | Account master + contract terms + joint holders + balances | ⚠️ Mixed — core account is correct; contract terms → Contract SA; joint holders → Party SA |
| `account_balance_summary_dd` | Daily balance summary | ✅ Correct |
| `account_eod_balance` | End-of-day balance (8.8 GB) | ✅ Correct |
| `currency_units_conversion_rate` | FX conversion rates | ❌ Belongs in `silver.reference` |
| `reference_currency_code` | ISO currency codes | ❌ Belongs in `silver.reference` |
| `financial_instrument` | Financial instruments | ❌ Belongs in `silver.treasury` or `silver.wealth` |
| `reference_financial_instrument_type` | Financial instrument types | ❌ Belongs in `silver.treasury` |
| `reference_product` | Product reference | ❌ Belongs in `silver.product` |
| `demand_draft` | Demand draft instruments | 🟡 Ambiguous — may belong in `silver.payment` |
| `reference_demand_draft_status` | DD status codes | 🟡 Ambiguous — follows `demand_draft` |
| `reference_demand_draft_stop_status` | DD stop-payment status | 🟡 Ambiguous |
| `banker_draft_unclaimed` | Unclaimed banker's drafts | 🟡 May belong in `silver.payment` |
| `dormant_account` | Dormant account flag | ✅ Acceptable account lifecycle sub-entity |
| `pbd_account_master` | Pre-banking-day account master (PBD prefix = source system prefix!) | ❌ Source-system prefix violates NAM-003 |
| `ccod_account_master_table` | Credit card OD account master (CCOD prefix = product type prefix!) | ❌ Source-system prefix; potential duplicate of `account_agreement_master` |
| `sbca_master_table` | SBCA = likely Small Business CASA account master | ❌ Source-system prefix; should be unified with core account entity |
| `current_bal_report` | Balance report | ❌ "Report" suffix implies aggregation — violates SLV-007 |
| `range_key_table` | Unknown (0 bytes) | ❌ Scope and ownership unclear; likely an ETL artifact |

**Finding:** 6–9 physical tables are incorrectly scoped into this SA. There are also 3 tables with source-system prefixes (`pbd_`, `ccod_`, `sbca_`) that should be unified under the canonical account entity.

### 2.2 Domain Decomposition vs. BIAN / IFW

**Expected BIAN Account structure:**
- **BIAN Service Domain: Current Account** → account master, balance, fees, interest
- **BIAN Service Domain: Term Deposits** → maturity, rollover
- **BIAN Service Domain: Savings Account** → CASA
- Account SA should host: Account (master), Account Balance (daily/snapshot), Account Status History

**Logical model vs. BIAN pattern:**

| BIAN Pattern | Present in LM? | Gap |
|---|---|---|
| Core account entity (account_sk, account_bk, type, status, open/close date) | ❌ Missing | No canonical `account` entity — `Account_Agreement_Master` partially substitutes but is over-fat |
| Balance entity (grain: per account per snapshot date) | Partial (`Account_Balance_Summary_DD`) | Exists but typed as STRING throughout; `account_eod_balance` has data but no logical model entry |
| Product association history | `Account_Product_Histroy` (typo) | Partial — product denormalized into account master |
| Status/lifecycle history | `Account_Renewal_Histroy` (typo) | Incomplete — renewal only, not full status lifecycle |
| Joint holder pattern | Flat columns in master | ❌ Violates 3NF — needs Party-Account child table |
| Exposure/collateral | `Account_ExposureAgreement`, etc. | ❌ Wrong SA — belongs in `silver.collateral` |
| Risk grading | `Account_Group_Risk_*` | ❌ Wrong SA — belongs in `silver.risk` |
| Access device | `Account_Access_Device` | ❌ Wrong SA — belongs in `silver.event` |

### 2.3 Identity Strategy (SLV-001 / SLV-002)

**Guideline SLV-001:** *"Every Silver entity must have `<entity>_sk` (MD5 surrogate) as primary key."*  
**Guideline SLV-002:** *"Every Silver entity must have `<entity>_bk` (business/natural key) as a non-nullable, unique, indexed column."*

**Findings:**
- **No logical entity in the Account SA declares an `account_sk` or `account_bk` column.** The logical model uses "Account Number" and "Account Modifier Number" as apparent composite natural keys, but these are source-system IDs named in plain English — not conforming to the `<entity>_bk` convention.
- The physical table `account_agreement_master` has `columnMapping.mode=name` and `maxColumnId=68`, implying 68 columns — but the logical model shows only 35 attributes. There is no DDL available to verify SK/BK presence physically.
- Several entities use `Account group id`, `Risk Grade Id`, `Mitigant id`, `Agreement id` as apparent PKs — none follow the `<entity>_sk` pattern.
- `CHAR(18)` used for some key fields in the logical model appears to be an Erwin modeling artifact (default char length) rather than a deliberate type decision.

**Severity: P0 / High — violates SLV-001 and SLV-002 on all entities.**

### 2.4 SCD Strategy (SLV-003)

**Guideline SLV-003:** *"SCD Type 2 is the default for all Silver entities. SCD Type 1 requires explicit data-steward approval."*

**Findings:**
- `account_agreement_master` has `changeDataFeed` enabled — suggests SCD-2 awareness, but the presence of `generatedColumns` (likely for `valid_from`/`valid_to`) needs physical DDL verification.
- `account_eod_balance` (8.8 GB, the main balance table) has **no `changeDataFeed`** enabled — likely append-only daily snapshots, which is acceptable for a daily balance grain but needs confirmation that `is_current` is maintained.
- 16 remaining physical tables show no explicit SCD-2 configuration.
- The logical model attributes include `Business Start Date` / `Business End Date` / `Is Active Flag` on most entities, which are the correct SCD-2 columns — **but they are all typed `CHAR(18)`**, which is incorrect (should be `TIMESTAMP` and `BOOLEAN`).
- `effective_from`, `effective_to`, `is_current` are listed as audit columns in the guideline but not explicitly declared in any entity's attribute list in the logical model.

**Severity: P0 / High — SCD-2 implementation is inconsistent and improperly typed across all entities.**

### 2.5 Cross-Entity Referential Integrity

| FK Relationship | Status | Gap |
|---|---|---|
| `account_agreement_master.account_number` → canonical `account.account_bk` | ❌ | No canonical `account` entity exists to reference |
| `account_agreement_master.product_id` → `silver.product.product_bk` | ❌ | Cross-SA reference not documented; product description denormalized into account |
| `account_agreement_master.customer_number_cif` → `silver.party.party_bk` | ❌ | CIF stored as flat attribute, no formal FK to Party SA |
| `account_balance_summary_dd.account_number` → account master | Partial | No formal FK constraint visible |
| `account_eod_balance.*` → any logical entity | ❌ | No logical entity defined for this physical table |
| Collateral entities → account | ❌ | Collateral entities should not be in this SA |
| Risk entities → account_group | Partial | Risk entities have `account_group_id` but no FK to `account_group` table |

### 2.6 ETL Mapping Completeness

**Mapping workbook summary (Overview1 sheet):**
- Total attributes mapped: 373
- Mapped: 302 (81%)
- Not Available: 24 (6.4%)
- Pending for Modelers: 15 (4%)
- Pending for RAK Confirmation: 8 (2.1%)
- Blank: 24 (6.4%)

**Critical unmapped attributes:**
- `Account maturity date status` — Not Available (critical for IFRS 9 staging)
- `Hold code` / `Hold code description` — Not Available (regulatory/compliance use)
- `Non-performing loan status` — Not Available (critical for IFRS 9/Basel)
- `Month to date interest` — Not Available
- `Asset Liability Code` — Not Available (Balance Sheet classification, critical for GL reconciliation)
- `Proposal Id` — Not Available (breaks traceability to Application SA)
- All `Reference_Data_Source_Code` attributes — "will be provided by DE" (delayed)
- All Reference_Account_Group_Type/Limit_Change_Reason/Related_Role_Code reference tables — Not Available

**ETL anomalies in mappings:**
- `Contract Name` mapped to `CRMUSER.ACCOUNTS.CUST_LAST_NAME` — semantically wrong: customer last name ≠ contract name
- `Contract Expiration Date` mapped to `CRMUSER.ACCOUNTS.BODATEMODIFIED` — semantically wrong: last modified date ≠ expiry date
- `Segment of primary account holder` mapped to `CRMUSER.ACCOUNTS.PRIMARYPERSONID` (a NUMBER ID) — wrong source column: ID ≠ segment
- `Available balance` and `Principal balance` both mapped to the same source column `CUSTOM.CUST_PROF_DATA.AVG_BAL_AED` — duplicate mapping, ambiguous
- `Account Group Loss Given Default Rate` mapped from empty table `TBAADM.MUD_PROFIT_CALC_DIST_TABLE` (remark: "Table is empty")
- Several `Account_Renewal_History` attributes mapped from `BLM_CONTRACTOR_DETAILS_TBL` (remark: "Table is empty")
- 1st/2nd joint account holders mapped from `TBAADM.SD_LOCK_CUSTOMER_MASTER` (remark: "Table empty") — critical gap for joint account regulatory reporting
- `Account Group Credit Limit Amount` — Not Available (credit limit history absent)
- `Risk Scenario Id` — Not Available on 3 risk entities
- Multiple risk metrics mapped to `RISK_PROFILE_SCORE` column when no actual risk metric type code exists — remark: "There is no Risk Metric code but we have Risk Profile Score"

---

## 3. Entity Inventory

| # | Logical Entity | Physical Table | Status | Logical Attrs | Mapped | Not Available | Priority |
|---|---|---|---|---|---|---|---|
| 1 | Account_Agreement_Master | `account_agreement_master` | ✅ Partial | 35 | ~28 | 7 | P0 |
| 2 | Account_Balance_Summary_DD | `account_balance_summary_dd` | ✅ Partial | 11 | 9 | 2 | P1 |
| 3 | Account_Account_Group | ❌ No physical table | 🔴 Unmapped | 14 | 6 | 1 | P1 |
| 4 | Account_Group | ❌ No physical table | 🔴 Unmapped | 17 | 7 | 1 | P1 |
| 5 | Account_Group_Bal_Hist_DD | ❌ No physical table | 🔴 Unmapped | 9 | 6 | 1 | P1 |
| 6 | Account_Group_Criterion | ❌ No physical table | 🔴 Unmapped | 6 | 1 | 1 | P2 |
| 7 | Account_Group_Feature | ❌ No physical table | 🔴 Unmapped | 5 | 1 | 4 | P2 |
| 8 | Account_Group_Limit_Hist | ❌ No physical table | 🔴 Unmapped | 9 | 4 | 2 | P1 |
| 9 | Account_Group_Related | ❌ No physical table | 🔴 Unmapped | 8 | 4 | 1 | P2 |
| 10 | Account_Group_Risk_Grade | ❌ No physical table | ❌ Wrong SA | 9 | 2 | 4 | P0 (relocate) |
| 11 | Account_Group_Risk_Hist | ❌ No physical table | ❌ Wrong SA | 13 | 4 | 4 | P0 (relocate) |
| 12 | Account_Group_Risk_Type_Hist | ❌ No physical table | ❌ Wrong SA | 9 | 4 | 1 | P0 (relocate) |
| 13 | Account_Group_Score_DD | ❌ No physical table | ❌ Wrong SA | 7 | 2 | 2 | P0 (relocate) |
| 14 | Account_Group_Type_Assoc | ❌ No physical table | 🔴 Unmapped | 7 | 4 | 0 | P2 |
| 15 | Account_Product_Histroy | ❌ No physical table | 🔴 Unmapped | 6 | 5 | 0 | P1 |
| 16 | Account_Renewal_Histroy | ❌ No physical table | 🔴 Unmapped | 5 | 3 | 0 | P1 |
| 17 | Account_Access_Device | ❌ No physical table | ❌ Wrong SA | 6 | 4 | 2 | P0 (relocate) |
| 18 | Account_Access_Device_Feature | ❌ No physical table | ❌ Wrong SA | 8 | 7 | 1 | P0 (relocate) |
| 19 | Agreement | ❌ No physical table | ❌ Wrong SA | 22 | 10 | 7 | P0 (relocate) |
| 20 | Account_ExposureAgreement | ❌ No physical table | ❌ Wrong SA | 21 | 7 | 6 | P0 (relocate) |
| 21 | Account_ExposureAssetCollateral | ❌ No physical table | ❌ Wrong SA | 21 | 8 | 5 | P0 (relocate) |
| 22 | Account_ExposureRiskMitigant | ❌ No physical table | ❌ Wrong SA | 18 | 8 | 4 | P0 (relocate) |
| 23 | Reference_Account_Agreement_Type | ❌ No physical table | 🔴 Unmapped | 9 | — | — | P1 |
| 24 | Reference_Account_Risk_Category | ❌ No physical table | ❌ Wrong SA | 9 | — | — | P0 (relocate) |
| 25 | Reference_Account_Type | ❌ No physical table | 🔴 Unmapped | 8 | 2 | 0 | P1 |
| 26 | Reference_Asset_Collateral_Type | ❌ No physical table | ❌ Wrong SA | 9 | — | — | P0 (relocate) |
| 27 | Reference_Collateral_Insurance_Status | ❌ No physical table | ❌ Wrong SA | 9 | — | — | P0 (relocate) |
| 28 | Reference_Collateral_Lien_Status | ❌ No physical table | ❌ Wrong SA | 9 | — | — | P0 (relocate) |
| 29 | Reference_Guarantor_Provider | ❌ No physical table | ❌ Wrong SA | 14 | — | — | P0 (relocate) |
| 30 | Reference_Mitigant_Type | ❌ No physical table | ❌ Wrong SA | 9 | — | — | P0 (relocate) |
| — | `account_eod_balance` (8.8 GB!) | Physical only | 🔴 No logical entity | — | — | — | **P0** |
| — | `currency_units_conversion_rate` | Physical only | ❌ Wrong SA | — | — | — | P0 (relocate) |
| — | `financial_instrument` | Physical only | ❌ Wrong SA | — | — | — | P0 (relocate) |
| — | `reference_financial_instrument_type` | Physical only | ❌ Wrong SA | — | — | — | P0 (relocate) |
| — | `reference_product` | Physical only | ❌ Wrong SA | — | — | — | P0 (relocate) |
| — | `reference_currency_code` | Physical only | ❌ Wrong SA | — | — | — | P0 (relocate) |
| — | `demand_draft` | Physical only | 🟡 Scope TBD | — | — | — | P1 |
| — | `pbd_account_master` | Physical only | ❌ Naming violation | — | — | — | P1 |
| — | `ccod_account_master_table` | Physical only | ❌ Naming violation | — | — | — | P1 |
| — | `sbca_master_table` | Physical only | ❌ Naming violation | — | — | — | P1 |
| — | `dormant_account` | Physical only | 🟡 Acceptable | — | — | — | P1 |
| — | `banker_draft_unclaimed` | Physical only | 🟡 Scope TBD | — | — | — | P2 |
| — | `current_bal_report` | Physical only | ❌ Aggregation/naming violation | — | — | — | P0 |
| — | `range_key_table` | Physical only | ❌ ETL artifact | — | — | — | P1 |

**Missing canonical entity (critical):** No `account` entity at all in the logical model. Per subject-areas.md, Account SA is supposed to host "Holding Containers: Unified Balance records." The central entity of this SA does not exist in the logical model.

---

## 4. Per-Entity Assessments

---

### 4.1 Entity: Account_Agreement_Master → `account_agreement_master`

#### 4.1a Industry Fitness

- **Boundary issue:** This entity conflates four distinct concerns: (1) account master/identity (account number, type, currency, open/close date, status), (2) balance metrics (principal balance, available balance, average balance, market value, outstanding amount, MTD amounts), (3) party information (primary account holder name, 1st/2nd joint holder, customer CIF, segment), and (4) product attributes (product ID, product description, account category, IBAN, Swift, routing).
- **3NF violation:** Balances (time-varying) stored alongside master (slowly-changing) attributes — different SCD grains mixed in one table.
- **BIAN pattern:** BIAN separates Current Account (master) from Current Account Balance (balance as of date). IFW has a distinct `ACCOUNT_BALANCE` entity.
- **Grain ambiguity:** The logical model definition says "when customer goes in an agreement with the financial institution" — this is agreement-grain, but the included balance columns imply a balance-snapshot grain.

#### 4.1b Attribute-Level Review

| Attribute | Logical Type | Source | Physical Col | Status | Issues |
|---|---|---|---|---|---|
| Account number | STRING | `CRMUSER.ACCOUNTS.ACCOUNTID` | — | Mapped | Should be `account_bk` (STRING NOT NULL UNIQUE); source column is `NUMBER` type cast to STRING without documented canonicalization |
| Account type | STRING | `CRMUSER.ACCOUNTS.ACCOUNTTYPE` | — | Mapped | Raw source code stored directly — violates SLV-004; should be `account_type_code` (canonical); nullable source |
| Contract type | STRING | `CRMUSER.ACCOUNTS.SUBSEGMENT` | — | Mapped | SUBSEGMENT ≠ contract type — semantic mismatch; should be in Contract SA |
| Currency code | STRING | `CRMUSER.ACCOUNTS.CRNCY_CODE` | — | Mapped | Naming: should be `currency_code` (ISO 4217); type correct as STRING |
| Product ID | STRING | `CRMUSER.SALES.PRODUCTID` → join | — | Mapped | Cross-table join introduces fan-out risk; product ID belongs in Account_Product_Histroy, not in master |
| Product description | STRING | `CRMUSER.PRODUCTS.PRODUCTNAME` | — | Mapped | ❌ Denormalization — product description must live in Product SA, not Account |
| Account open date | STRING | `CRMUSER.ACCOUNTS.RELATIONSHIPOPENINGDATE` | — | Mapped | ❌ Type should be `DATE`, not `STRING`; column name: `account_open_date` ✓ convention but typed wrong |
| Account close date | STRING | `TBAADM.GENERAL_ACCT_MAST_TABLE.ACCT_CLS_DATE` | — | Mapped | ❌ Type: `DATE` not `STRING` |
| Account value date | STRING | `CRMUSER.ACCOUNTS.STARTDATE` | — | Mapped | ❌ Type: `DATE`; naming: `account_value_date` ✓ |
| Account maturity date | STRING | None | — | **Not Available** | ❌ Critical gap for IFRS 9 ECL staging — maturity is required for EAD calculation |
| Non-performing loan status | STRING | None | — | **Not Available** | ❌ Critical gap for BCBS 239 / IFRS 9; should be in Loan SA but cross-cutting ref needed |
| Primary account holder | STRING | `CRMUSER.ACCOUNTS.NAME` | — | Mapped | ❌ PII — name must be SHA-256 masked (DQ guideline §4 PII); should be `party_bk` FK, not embedded name |
| 1st joint account holder | STRING | `TBAADM.SD_LOCK_CUSTOMER_MASTER.JOINT_HOLDER_NAME_1` | — | Mapped (table empty) | ❌ 3NF violation — repeating group; ❌ PII masking required; source table empty |
| 2nd joint account holder | STRING | `TBAADM.SD_LOCK_CUSTOMER_MASTER.JOINT_HOLDER_NAME_2` | — | Mapped (table empty) | ❌ Same as above |
| Month to date accrued amount | STRING | `TBAADM.GENERAL_ACCT_MAST_TABLE.CUM_CR_AMT` | — | Mapped | ❌ Aggregation (MTD) in Silver — violates SLV-007; type should be `DECIMAL(18,4)` |
| Month to date balance | STRING | `CRMUSER.ACCOUNTS.CURRENTCREXPOSURE` | — | Mapped | ❌ Same — MTD aggregation violates SLV-007 |
| Month to date interest | STRING | None | — | Not Available | ❌ MTD aggregation — should not be in Silver anyway |
| Bonus accrued MTD days | STRING | None | — | Not Available | ❌ MTD aggregation — violates SLV-007 |
| Segment of primary account holder | STRING | `CRMUSER.ACCOUNTS.PRIMARYPERSONID` | — | Mapped | ❌ Semantic mismatch: PRIMARYPERSONID (a NUMBER ID) ≠ segment code; cross-SA denormalization (segment is Party attribute) |
| Principal balance in AED | STRING | `CUSTOM.CUST_PROF_DATA.AVG_BAL_AED` | — | Mapped | ❌ Type: `DECIMAL(18,4)`; naming: `principal_balance_amount_aed`; duplicate mapping — same source col as available balance |
| Available balance in AED | STRING | `CUSTOM.CUST_PROF_DATA.AVG_BAL_AED` | — | Mapped | ❌ Same source column as principal balance — clear mapping error |
| Average balance in AED | STRING | `CUSTOM.C_ACCPROFILE.AVG_ACCOUNT_VALUE_AMOUNT` | — | Mapped | ❌ Aggregation in Silver violates SLV-007; type should be `DECIMAL(18,4)` |
| Market value in AED | STRING | `CRMUSER.PRODUCTCURRENCY.COSTBASIS` | — | Mapped | ❌ Belongs in Wealth/Treasury SA if market value; nullable source |
| Mark-to-market in AED | STRING | `CRMUSER.SALEDMAT.ACCMARKETVALUE` | — | Mapped | ❌ MTM is a derived value — violates SLV-006/SLV-007; should be in Wealth SA |
| Outstanding amount in AED | STRING | `CRMUSER.SALECCOD.ACCOUTSTANDINGAMOUNT` | — | Mapped | ❌ If CC outstanding, belongs in Card SA |
| Account category code | STRING | `CRMUSER.ACCOUNTS.ACCOUNTTYPE` | — | Mapped | Duplicate source: same col as `account type` above — ambiguous |
| Hold code | STRING | None | — | **Not Available** | ❌ Regulatory freeze/hold codes absent — CBUAE compliance gap |
| Hold code description | STRING | None | — | Not Available | Same |
| Principal balance in local currency | STRING | None | — | Not Available | ❌ Local currency amounts missing — FX reporting gap |
| KYC Status | STRING | None (not in excerpt, but mentioned in LM definition) | — | Ambiguous | ❌ KYC belongs in `silver.party` / `silver.risk`, not Account SA |
| IBAN number | STRING | — | — | Mapped | ❌ IBAN should be a payment identity attribute; potentially in `silver.payment` |
| Swift code | STRING | — | — | Mapped | ❌ Swift is payment infrastructure — `silver.payment` |
| Banking routing code | STRING | — | — | Mapped | Same |
| Customer number (CIF) | CHAR(18) | — | — | Mapped | Should be `party_bk` (FK to Party SA); CHAR(18) data type incorrect |
| Bank ID | CHAR(18) | — | — | Mapped | Should reference `silver.org` |
| Regulatory body | CHAR(18) | — | — | Mapped | Reference data — should reference `silver.reference` |
| Business Start Date | CHAR(18) | — | — | — | ❌ SCD-2 column typed as CHAR(18) — must be `TIMESTAMP` |
| Business End Date | CHAR(18) | — | — | — | ❌ Same |
| Is Active Flag | CHAR(18) | — | — | — | ❌ Should be `BOOLEAN` (`is_current`), not CHAR(18) |

#### 4.1c Metadata Completeness

Per guideline §2.7 of 11-modeling-dos-and-donts.md, required metadata columns:

| Metadata Column | Required Type | Present in LM? | Gap |
|---|---|---|---|
| `source_system_code` | STRING | ✅ Yes ("Source system code") | OK |
| `source_system_id` | STRING | ✅ Yes ("Source system id") | OK |
| `create_date` | TIMESTAMP | ✅ Yes ("Create date") — typed STRING | ❌ Type incorrect |
| `update_date` | TIMESTAMP | ✅ Yes ("Update date") — typed STRING | ❌ Type incorrect |
| `delete_date` | TIMESTAMP | ✅ Yes ("Deleted date") — typed STRING | ❌ Type incorrect |
| `is_active_flag` | STRING | ✅ Yes ("Is Active Flag") — typed CHAR(18) | ❌ Type incorrect |
| `effective_from` | TIMESTAMP | Approximated by "Business Start Date" CHAR(18) | ❌ Wrong name + type |
| `effective_to` | TIMESTAMP | Approximated by "Business End Date" CHAR(18) | ❌ Wrong name + type |
| `is_current` | BOOLEAN | Not explicitly present | ❌ Missing |
| `run_id` | STRING | Not present | ❌ Missing |
| `source_ingestion_date` | TIMESTAMP | Not present | ❌ Missing |
| `IsLogicallyDeleted` | STRING | Present | ❌ Non-standard name; should be `is_deleted` (BOOLEAN) |
| `record_hash` | STRING | Not present | ❌ Missing (DQ controls guideline) |
| `data_quality_status` | STRING | Not present | ❌ Missing (DQ controls guideline) |
| `dq_issues` | STRING | Not present | ❌ Missing (DQ controls guideline) |

**Score: 4 of 14 required metadata columns present with correct name; all typed incorrectly or missing entirely.**

---

### Finding Account_Agreement_Master-001 — Missing Canonical account_sk / account_bk

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | `SLV-001` — "Every Silver entity must have `<entity>_sk` (MD5 surrogate) as primary key"; `SLV-002` — "Every Silver entity must have `<entity>_bk` business key" |
| Evidence | Logical model: `Account_Agreement_Master` has 35 attributes, none named `account_sk` or `account_bk`. Source key is `ACCOUNTID (NUMBER)` — source-system ID |
| Affected Table | `silver.account.account_agreement_master` |
| Affected Column(s) | All — PK is implicit on source `ACCOUNTID` |
| Confidence | 0.97 |

**Description:** The entity has no `account_sk` (MD5 deterministic surrogate) nor `account_bk` (canonicalized business key). The source column `ACCOUNTID` (NUMBER) is used as an implicit natural key. This means Gold-layer FK references cannot be stable across re-ingestion events, breaks idempotency, and violates SLV-001/002.
**Remediation:** Add `account_sk STRING NOT NULL` (derived as `MD5(UPPER(TRIM(account_bk)))`) and `account_bk STRING NOT NULL` (canonicalized from `ACCOUNTID`) as the first two columns; define `CONSTRAINT pk_account_agreement_master PRIMARY KEY (account_sk)`.
**Estimated Effort:** Medium  
**Owner:** Data Modeling / ETL

---

### Finding Account_Agreement_Master-002 — All Attributes Typed as STRING (Including Monetary and Date)

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | DQ Controls §3.1 — "All monetary columns must use `DECIMAL(18,4)`"; DQ Controls §3.1 Standardisation — "All date columns must conform to `YYYY-MM-DD` format"; NAM-003 convention `<measure>_amount_<currency>` |
| Evidence | Logical model: all 35 attributes typed STRING or CHAR(18). Source types include NUMBER(22), DATE, NUMBER(20,4) — all explicitly typed but target typed STRING. |
| Affected Table | `silver.account.account_agreement_master` |
| Affected Column(s) | `account_open_date`, `account_close_date`, `account_value_date`, `principal_balance_*`, `available_balance_*`, `average_balance_*`, `outstanding_amount_*`, `credit_limit`, `interest_rate`, all monetary/date columns |
| Confidence | 0.99 |

**Description:** Every attribute is mapped as STRING including date fields (should be DATE/TIMESTAMP) and monetary fields (should be DECIMAL(18,4) at Silver; BIGINT in fils at lowest-unit layer per SLV-010). This is a critical data precision risk — monetary precision is lost, date arithmetic is impossible, and downstream Gold layer type inference will fail.
**Remediation:** Re-define all logical data types: dates → `DATE`; timestamps → `TIMESTAMP`; monetary amounts → `DECIMAL(18,4)` with `_amount_aed` suffix; interest rates → `DECIMAL(10,6)`; counts → `BIGINT`; boolean indicators → `BOOLEAN`. Regenerate physical DDL from corrected logical model.
**Estimated Effort:** Large  
**Owner:** Data Modeling

---

### Finding Account_Agreement_Master-003 — Aggregated (MTD) Metrics in Silver Entity

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | `SLV-007` — "No pre-computed metrics"; `SLV-006` — "3NF; no derived values/aggregations" |
| Evidence | Logical model includes: "month to date accrued amount", "month to date balance", "month to date interest", "month to date accrued amount in AED", "month to date balance in AED", "month to date interest in AED", "bonus accrued in AED" — all MTD aggregations |
| Affected Table | `silver.account.account_agreement_master` |
| Affected Column(s) | 7 MTD/aggregated columns |
| Confidence | 0.98 |

**Description:** Month-to-date (MTD) amounts are pre-computed aggregations of transactional data. Silver must contain only atomic, un-aggregated facts. These columns must be removed from Silver and computed in the Gold layer from atomic transaction records in `silver.transaction`.
**Remediation:** Remove all MTD-calculated columns from `Account_Agreement_Master`. In Gold, compute `fact_account_monthly_balance` by aggregating from `silver.transaction.financial_transaction` with monthly grain.
**Estimated Effort:** Medium  
**Owner:** Data Modeling / ETL

---

### Finding Account_Agreement_Master-004 — Joint Account Holders as Repeating Columns (3NF Violation)

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | High |
| Guideline Rule | `SLV-006` — "3NF; no derived values/aggregations"; guideline 11-modeling-dos-and-donts §2.1 — "Repeating groups must become child tables" |
| Evidence | Logical model attributes: "primary account holder", "1st joint account holder", "2nd joint account holder" — three fixed positions for an inherently variable-length list. Source tables for joint holders are empty. |
| Affected Table | `silver.account.account_agreement_master` |
| Affected Column(s) | `primary_account_holder`, `1st_joint_account_holder`, `2nd_joint_account_holder` |
| Confidence | 0.97 |

**Description:** Storing joint account holders as numbered columns (`1st`, `2nd`) is a repeating group — a 1NF/3NF violation. A customer may have more than 2 joint holders. More critically, these are party names (PII), not party FK references — they should reference `silver.party.party_bk`. The source tables for joint holder data are empty, meaning joint account regulatory reporting is currently broken.
**Remediation:** Create a child entity `account_party_role` (account_party_role_sk, account_bk, party_bk, party_role_type_code [PRIMARY/JOINT/BENEFICIAL], role_start_date, role_end_date, + metadata). Remove the three flat columns from `Account_Agreement_Master`.
**Estimated Effort:** Medium  
**Owner:** Data Modeling

---

### Finding Account_Agreement_Master-005 — Party Attributes Denormalized into Account Entity

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | High |
| Guideline Rule | `SLV-006` — "3NF"; guideline 11-modeling-dos-and-donts §2.2 — "Subject-Area Isolation" |
| Evidence | Logical model includes: "primary account holder" (party name), "segment of primary account holder" (mapped to PRIMARYPERSONID — a Party attribute), "KYC Status" (party compliance attribute), "customer number (CIF)" embedded in account entity |
| Affected Table | `silver.account.account_agreement_master` |
| Affected Column(s) | `primary_account_holder`, `segment`, `kyc_status`, `customer_number_cif` |
| Confidence | 0.93 |

**Description:** Party identity attributes (name, segment, KYC) belong in `silver.party`. Embedding them in Account creates update anomalies: when a customer's segment changes, every account row must be updated. `customer_number_cif` should be a FK reference `party_bk`, not an embedded attribute.
**Remediation:** Remove party attributes from `Account_Agreement_Master`. Keep only `party_bk` as an FK reference. Gold layer joins `silver.account.account_agreement_master` with `silver.party.party` to enrich.
**Estimated Effort:** Medium  
**Owner:** Data Modeling

---

### Finding Account_Agreement_Master-006 — Product Description Denormalized into Account Entity

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | Medium |
| Guideline Rule | `SLV-006` — "3NF; no derived values/aggregations"; §2.2 Subject-Area Isolation |
| Evidence | Mapping: "product description" mapped from `CRMUSER.PRODUCTS.PRODUCTNAME` via a 3-table join — introduces fan-out risk and violates SA isolation |
| Affected Table | `silver.account.account_agreement_master` |
| Affected Column(s) | `product_description` |
| Confidence | 0.95 |

**Description:** Product name is a Product SA attribute. It will change across product versions. Storing it in Account creates update anomalies and creates a cross-SA Silver join at ingestion time (violates §2.2).
**Remediation:** Remove `product_description` from Account. Keep `product_bk` as FK. Join at Gold.
**Estimated Effort:** Small  
**Owner:** Data Modeling

---

### Finding Account_Agreement_Master-007 — Incorrect Source Column Mappings (Semantic Errors)

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | ETL Mapping completeness; data accuracy per DQ Controls §3.1 |
| Evidence | Mapping file: "Contract Name" → `CRMUSER.ACCOUNTS.CUST_LAST_NAME`; "Contract Expiration Date" → `CRMUSER.ACCOUNTS.BODATEMODIFIED`; "Segment" → `CRMUSER.ACCOUNTS.PRIMARYPERSONID`; "Available balance" and "Principal balance" both → `CUSTOM.CUST_PROF_DATA.AVG_BAL_AED` |
| Affected Table | `silver.account.account_agreement_master` |
| Affected Column(s) | `contract_name`, `contract_expiration_date`, `segment`, `available_balance_amount_aed`, `principal_balance_amount_aed` |
| Confidence | 0.97 |

**Description:** Four confirmed semantic mapping errors: (1) customer last name used as contract name — will produce garbage data; (2) record last-modified date used as contract expiry — regulatory reporting risk; (3) person numeric ID used as segment — type and semantic mismatch; (4) average balance column used for both principal and available balance — will produce identical, incorrect values.
**Remediation:** (1) Identify correct source for contract name (likely BLM_CONTRACTOR_DETAILS_TBL.CONTRACTOR_NAME or a contract-specific table); (2) Identify maturity/expiry date source; (3) Map segment via CIF join to CRMUSER.ACCOUNTS.SUBSEGMENT or SEGMENT; (4) Identify distinct source columns for principal vs available balance.
**Estimated Effort:** Medium  
**Owner:** ETL

---

### Finding Account_Agreement_Master-008 — Payment Identity Attributes (IBAN, SWIFT, Routing) Misplaced

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | Medium |
| Guideline Rule | §2.2 Subject-Area Isolation; SA taxonomy — Payments SA owns `silver.payment` |
| Evidence | Logical model includes: "IBAN number", "Swift code", "Banking routing code" in Account_Agreement_Master |
| Affected Table | `silver.account.account_agreement_master` |
| Affected Column(s) | `iban_number`, `swift_code`, `banking_routing_code` |
| Confidence | 0.88 |

**Description:** IBAN and SWIFT/routing codes are payment network identities. While an account has an IBAN, the payment addressing attributes are best managed in `silver.payment` or as a child `account_payment_identifier` entity to avoid update anomalies. IBAN is also a BCBS 239 Critical Data Element.
**Remediation:** Create `account_identifier` child entity (account_identifier_sk, account_bk, identifier_type_code [IBAN/SWIFT/ROUTING], identifier_value, is_primary, valid_from, valid_to) or relocate to Payment SA.
**Estimated Effort:** Small  
**Owner:** Data Modeling

---

### Finding Account_Agreement_Master-009 — Missing Mandatory Metadata Columns

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | `SLV-008` — "All metadata columns present"; guideline 11-modeling-dos-and-donts §2.7 |
| Evidence | Logical model: `is_current` (BOOLEAN), `run_id`, `source_ingestion_date`, `record_hash`, `data_quality_status`, `dq_issues` are absent. Existing SCD columns (`Business Start Date`, `Business End Date`, `Is Active Flag`) are typed as `CHAR(18)` |
| Affected Table | `silver.account.account_agreement_master` |
| Affected Column(s) | `is_current`, `run_id`, `source_ingestion_date`, `record_hash`, `data_quality_status`, `dq_issues` |
| Confidence | 0.99 |

**Description:** 6 mandatory metadata columns are absent. Existing SCD columns use incorrect names (`Business Start Date` instead of `effective_from`) and incorrect types (`CHAR(18)` instead of `TIMESTAMP`/`BOOLEAN`). Without `is_current`, SCD-2 queries cannot function. Without `record_hash`, deduplication is not possible. Without `data_quality_status`, DQ-gated writes cannot be enforced.
**Remediation:** Add all missing columns with correct types. Rename `Business Start Date` → `effective_from TIMESTAMP`, `Business End Date` → `effective_to TIMESTAMP`, `Is Active Flag` → `is_current BOOLEAN`. Add `run_id STRING`, `source_ingestion_date TIMESTAMP`, `record_hash STRING`, `data_quality_status STRING`, `dq_issues STRING`.
**Estimated Effort:** Medium  
**Owner:** Data Modeling / DBA

---

### 4.2 Entity: Account_Balance_Summary_DD → `account_balance_summary_dd`

#### 4.2a Industry Fitness

- Grain definition ("DD" = Daily) is correct for a daily balance summary.
- Per BIAN, balance snapshots should be separate from the account master — this decomposition is correct.
- However, "Account Balance Summary Count" (a count of transactions in the period) is an aggregation metric and violates SLV-007.
- "Account Balance Summary Rate" (interest rate?) should be typed as DECIMAL, not STRING.

#### 4.2b Attribute-Level Review

| Attribute | Logical Type | Source | Status | Issues |
|---|---|---|---|---|
| Account Number | STRING | `TBAADM.GENERAL_ACCT_MAST_TABLE.ACCT_NUM` | Mapped | Should be `account_bk` FK reference |
| Account Modifier Number | STRING | `TBAADM.GENERAL_ACCT_MAST_TABLE.ACID` | Mapped | Composite key part — needs normalization |
| Balance Category Type Code | STRING | `CRMUSER.CATEGORIES.CATEGORYTYPE` → transform | Mapped | ✅ Uses code — check against `silver.reference.code_mapping` |
| Account Metric Type Code | STRING | `TBAADM.GENERAL_ACCT_MAST_TABLE.SCHM_TYPE` | Mapped | Raw source code — SLV-004 check needed |
| Account Balance Summary Start Dttm | STRING | `TBAADM.EOD_ACCT_BAL_TABLE.EOD_DATE` | Mapped | ❌ Type: should be `DATE` named `balance_start_date` |
| Account Balance Summary End Dttm | STRING | `TBAADM.EOD_ACCT_BAL_TABLE.END_EOD_DATE` | Mapped | ❌ Type: should be `DATE` named `balance_end_date` |
| Account Balance Summary Tm Pd Code | STRING | None | Not Available | Period code absent — analytical usability impacted |
| Account Balance Summary Amount | STRING | `TBAADM.EOD_ACCT_BAL_TABLE.TRAN_DATE_BAL` (NUMBER(20,4)) | Mapped | ❌ Type: `DECIMAL(18,4)`; naming: `balance_amount_aed` |
| Account Currency Balance Summary Amount | STRING | `TBAADM.EOD_ACCT_BAL_TABLE.EAB_CRNCY_CODE` | Mapped | ❌ Semantic error: column stores currency code (VARCHAR3), not an amount! |
| Account Balance Summary Rate | STRING | `TBAADM.TD_ACCT_MASTER_TABLE.ACCT_CLOSE_INTEREST_RATE` | Mapped | ❌ Type: `DECIMAL(10,6)` named `interest_rate` |
| Account Balance Summary Count | STRING | `TBAADM.TD_ACCT_MASTER_TABLE.TS_CNT` | Mapped | ❌ Aggregation metric in Silver — violates SLV-007 |

#### 4.2c Metadata Completeness

No metadata columns at all visible in the logical model for this entity. The same issues as Account_Agreement_Master apply.

---

### Finding Account_Balance_Summary_DD-001 — Semantic Error: Currency Code Column Named as Amount

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | Data accuracy / ETL mapping correctness |
| Evidence | Mapping: "Account Currency Balance Summary Amount" mapped to `EOD_ACCT_BAL_TABLE.EAB_CRNCY_CODE VARCHAR2(3)` — a 3-char currency code column |
| Affected Table | `silver.account.account_balance_summary_dd` |
| Affected Column(s) | `account_currency_balance_summary_amount` |
| Confidence | 0.98 |

**Description:** The attribute is named "Amount" but is mapped to a currency code column. This will store "AED", "USD" etc. in a column consumers will interpret as a monetary value. This is a critical data quality defect that will corrupt downstream balance calculations.
**Remediation:** Rename to `balance_currency_code STRING` and map to the currency code correctly. Separately identify the actual local-currency balance source column (likely `EAB_CRNCY_BAL` or similar in EOD_ACCT_BAL_TABLE) and add a `balance_amount_orig DECIMAL(18,4)` column.
**Estimated Effort:** Small  
**Owner:** ETL

---

### Finding Account_Balance_Summary_DD-002 — Count Metric Is an Aggregation (SLV-007)

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | Medium |
| Guideline Rule | `SLV-007` — "No pre-computed metrics"; `SLV-006` — "No derived values/aggregations" |
| Evidence | Mapping: "Account Balance Summary Count" → `TBAADM.TD_ACCT_MASTER_TABLE.TS_CNT NUMBER(5)` |
| Affected Table | `silver.account.account_balance_summary_dd` |
| Affected Column(s) | `account_balance_summary_count` |
| Confidence | 0.90 |

**Description:** A count of terms/transactions is an aggregation. Silver must not contain pre-computed metrics. This should be calculated in Gold from atomic transaction records.
**Remediation:** Remove from Silver. Compute in Gold layer.
**Estimated Effort:** Small  
**Owner:** Data Modeling

---

### 4.3 Entity: account_eod_balance (Physical Only — No Logical Entity)

#### Industry Fitness
This is the largest and most significant physical table (8.85 GB, the primary financial balance store). Its absence from the logical model is a critical governance gap. It appears to be the actual operational source for CBUAE regulatory reporting, IFRS 9 staging, and balance sheet reconciliation.

---

### Finding account_eod_balance-001 — No Logical Model Entity (Critical Governance Gap)

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | Guideline 11-modeling-dos-and-donts §2.11 — "Create and approve the logical and physical model before DDL; tables without a logical entity entry are a governance gap" |
| Evidence | Physical: `silver_dev_v2.account.account_eod_balance` exists (8.85 GB, 2 files, last modified 2026-03-22). Logical model: No `account_eod_balance` entity present in `bank_logical_model.xlsx` Account subject area |
| Affected Table | `silver.account.account_eod_balance` |
| Affected Column(s) | All columns |
| Confidence | 0.99 |

**Description:** The bank's primary end-of-day balance table (8.85 GB) has no approved logical model entry, no registered data steward, no DQ rules, and no documented lineage. This table likely feeds CBUAE regulatory balance sheet reports and IFRS 9 ECL staging. Operating a critical financial table outside the governance framework is a BCBS 239 P1 finding.
**Remediation:** Immediately register a logical entity `Account_EOD_Balance` in the Erwin model with full attribute definitions, data types, grain statement, and SCD strategy. Register the table in GDGC/data catalog with data steward, classification, and lineage. Apply all mandatory metadata columns. Define DQ rules for balance amount non-null, account_bk FK referential integrity.
**Estimated Effort:** Medium  
**Owner:** Data Modeling + Governance

---

### 4.4 Entity: Agreement → No Physical Table

#### 4.4a Industry Fitness

The `Agreement` entity — defined as "a formal arrangement or contract between parties outlining terms, conditions, and responsibilities" — is misplaced in `silver.account`. Per the subject-area taxonomy:
- **Contract SA** (`silver.contract`): "Legal Agreements: Account Opening Contracts, Loan Agreements, Card Member Agreements, and Service Agreements. Stores agreed rates, terms, and signatures."

The `Agreement` entity is definitionally a Contract SA entity. Furthermore, its attributes (`Account_Source_Code`, `External_Agreement_Ind`, `Proposal_Id`) show it straddles Application, Contract, and Account concerns.

---

### Finding Agreement-001 — Entity in Wrong Subject Area

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | SA taxonomy `sa/subject-areas.md` — Contract SA owns legal agreements; §2.2 Subject-Area Isolation |
| Evidence | Logical model entity `Agreement` in Account SA; definition matches Contract SA description exactly |
| Affected Table | No physical table yet — no migration risk, only model correction needed |
| Affected Column(s) | Entire entity |
| Confidence | 0.93 |

**Description:** The `Agreement` entity belongs in `silver.contract`. Placing it in `silver.account` will cause downstream consumers to look in the wrong schema and will create a precedent for mixing contract-layer concerns into the account layer.
**Remediation:** Move `Agreement` entity to `silver.contract` schema in the Erwin model. Map to a physical table `silver.contract.agreement`. Remove from Account SA.
**Estimated Effort:** Small (model change only — no physical table to migrate)  
**Owner:** Data Modeling

---

### 4.5 Collateral / Exposure / Risk Entities (Account_ExposureAgreement, Account_ExposureAssetCollateral, Account_ExposureRiskMitigant, Reference_Asset_Collateral_Type, Reference_Collateral_Insurance_Status, Reference_Collateral_Lien_Status, Reference_Guarantor_Provider, Reference_Mitigant_Type)

All 8 entities are misplaced. Per subject-areas.md: "Collateral — `silver.collateral` — Security pledged against lending: Real Estate, Metals, Cash, Securities. Asset Valuations, LTV Ratios, and Lien/Charge Registrations."

---

### Finding Collateral-001 — 8 Collateral/Exposure Entities in Wrong SA

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | `sa/subject-areas.md` §8b — Collateral SA owns pledged asset and exposure entities; §2.2 Subject-Area Isolation |
| Evidence | Logical model Account SA includes: `Account_ExposureAgreement`, `Account_ExposureAssetCollateral`, `Account_ExposureRiskMitigant`, `Reference_Asset_Collateral_Type`, `Reference_Collateral_Insurance_Status`, `Reference_Collateral_Lien_Status`, `Reference_Guarantor_Provider`, `Reference_Mitigant_Type` |
| Affected Table | No physical tables yet |
| Affected Column(s) | Entire entities |
| Confidence | 0.95 |

**Description:** Collateral, risk mitigant, exposure, and guarantor entities belong in `silver.collateral` (part of the Lending SA hierarchy). Their presence in Account SA will fragment collateral data governance and break the IFRS 9 collateral coverage calculation which requires all collateral data in one SA.
**Remediation:** Relocate all 8 entities to `silver.collateral` (or `silver.lending.collateral`). Update Erwin model. The `Account_ExposureAgreement.agreement_id` FK should reference `silver.contract.agreement`.
**Estimated Effort:** Small (model relocation only)  
**Owner:** Data Modeling + Credit Risk Steward

---

### 4.6 Risk Entities (Account_Group_Risk_Grade, Account_Group_Risk_Hist, Account_Group_Risk_Type_Hist, Account_Group_Score_DD, Reference_Account_Risk_Category)

---

### Finding RiskEntities-001 — 5 Risk Entities in Wrong SA

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | `sa/subject-areas.md` §17 — Risk & Compliance SA (`silver.risk`) owns "KYC Levels, AML Risk Scores, Watchlist Hits, Customer Risk Ratings, FATCA/CRS flags, and Basel RWA" |
| Evidence | Logical model Account SA includes risk grading, risk history, risk type history, and scoring entities |
| Affected Table | No physical tables yet |
| Affected Column(s) | Entire entities |
| Confidence | 0.93 |

**Description:** Credit risk grades, probability of default rates, LGD rates, and capital requirement amounts are Basel/IFRS 9 risk metrics that belong in `silver.risk`. Storing them in `silver.account` creates a fragmented risk data architecture that will fail BCBS 239 aggregation requirements.  
Additionally: `Account_Group_Risk_Type_Hist` has 4 risk metric attributes all mapped to the same source column `RISK_PROFILE_SCORE` — suggesting the risk framework for these entities doesn't yet exist in the source system.
**Remediation:** Relocate to `silver.risk`. Work with Risk steward to identify proper PD, LGD, EAD source data (likely from a risk engine or ECL model output, not CRMUSER).
**Estimated Effort:** Small (model relocation) + Medium (finding proper data sources)  
**Owner:** Data Modeling + Risk & Compliance Data Steward

---

### 4.7 Access Device Entities (Account_Access_Device, Account_Access_Device_Feature)

---

### Finding AccessDevice-001 — Device Access Entities in Wrong SA

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | Medium |
| Guideline Rule | `sa/subject-areas.md` §11 — Business Event SA owns "Logins, … POS Events. Includes Channel and Location keys" |
| Evidence | Logical model: `Account_Access_Device` and `Account_Access_Device_Feature` sourced from `TBAADM.LOGIN_LOG_TABLE` and `LOGIN_TABLE` — these are channel event data |
| Affected Table | No physical tables |
| Affected Column(s) | Entire entities |
| Confidence | 0.87 |

**Description:** Device access and feature tracking (login device, login time) are behavioral/event data — they belong in `silver.event` (Business Event SA), not `silver.account`. Placing them in Account SA will pollute the account entity with high-volume, time-series event data at a different grain.
**Remediation:** Relocate to `silver.event` as `device_access_event` and `device_feature_event` entities.
**Estimated Effort:** Small  
**Owner:** Data Modeling

---

### 4.8 Account_Group and Sub-Entities (Account_Group, Account_Group_Criterion, Account_Group_Feature, Account_Group_Limit_Hist, Account_Group_Related, Account_Group_Type_Assoc, Account_Account_Group)

#### 4.8a Industry Fitness

- `Account_Group` is a legitimate Account SA entity — it groups accounts for analytical and operational purposes (e.g., a customer's portfolio of accounts, a corporate group's accounts).
- `Account_Group_Limit_Hist` — credit limits belong here as account-group level credit limits. ✅
- `Account_Group_Related` — inter-group relationships are acceptable in Account SA.
- `Account_Group_Limit_Hist.Limit_Type_Code` mapped to `CRMUSER.ACCOUNTS.AVAILABLECRLIMIT` (a credit limit number, not a limit type code) — semantic mismatch.
- `Account_Group_Limit_Hist.Account_Group_Credit_Limit_Amount` — Not Available. Critical for credit risk and CBUAE regulatory capital reporting.

---

### Finding AccountGroup-001 — Account_Group Has No Physical Table Despite Being in Scope

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | High |
| Guideline Rule | Entity implementation completeness |
| Evidence | Logical model: `Account_Group` (17 attrs), `Account_Account_Group` (14 attrs) defined and mapped. Physical structures: no `account_group` or `account_account_group` table exists |
| Affected Table | Missing |
| Affected Column(s) | All |
| Confidence | 0.97 |

**Description:** Account groups (customer portfolio groupings) are required for multi-account credit limit management, group reporting, and CBUAE consolidated exposure calculation. Their absence means these use cases cannot be served from the Silver layer.
**Remediation:** Build physical tables `silver.account.account_group` and `silver.account.account_account_group` from the logical model, with proper SK/BK, SCD-2 columns, and all mandatory metadata. Fix the `Limit_Type_Code` mapping semantic error.
**Estimated Effort:** Medium  
**Owner:** ETL + Data Modeling

---

### 4.9 Account_Product_Histroy (Typo) → No Physical Table

---

### Finding AccountProduct-001 — Typo in Entity Name

| Field | Value |
|---|---|
| Priority | P2 |
| Criticality | Low |
| Guideline Rule | `NAM-001` — snake_case, no abbreviations; accurate naming |
| Evidence | Logical model entity name: `Account_Product_Histroy` (misspelling of "History") |
| Affected Table | Would be `silver.account.account_product_history` |
| Affected Column(s) | Entity name |
| Confidence | 0.99 |

**Description:** Typo in entity name. Physical table if ever created as `account_product_histroy` would violate naming standards.
**Remediation:** Rename to `Account_Product_History` in Erwin model.
**Estimated Effort:** Trivial  
**Owner:** Data Modeling

---

### Finding AccountProduct-002 — No Physical Table

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | Medium |
| Guideline Rule | Entity implementation completeness |
| Evidence | Logical model entity exists; physical structures: no `account_product_history` table |
| Affected Table | Missing |
| Affected Column(s) | All |
| Confidence | 0.95 |

**Description:** Account product history (which product was associated with an account and when) is needed for product switch reporting and fee recalculation. Currently absent from physical layer.
**Remediation:** Build `silver.account.account_product_history` table.
**Estimated Effort:** Small  
**Owner:** ETL

---

### 4.10 Account_Renewal_Histroy (Typo) → No Physical Table

---

### Finding AccountRenewal-001 — Typo + Source Tables Empty + Partial Mapping

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | Medium |
| Guideline Rule | `NAM-001`; mapping completeness |
| Evidence | Logical model entity: `Account_Renewal_Histroy`; mapping: all 3 date attributes mapped to `BLM_CONTRACTOR_DETAILS_TBL` (remark: "Table is empty") |
| Affected Table | Missing |
| Affected Column(s) | `contract_start_dttm`, `contract_term_expiration_dttm`, `contract_grace_period_end_dttm` |
| Confidence | 0.95 |

**Description:** Renewal history captures maturity/rollover events. The source table is empty in the source system. This entity cannot be populated currently. Additionally the entity name contains a typo.
**Remediation:** (1) Fix typo to `Account_Renewal_History`; (2) Identify alternative source for account maturity/rollover events (possibly `TBAADM.TD_ACCT_MASTER_TABLE` or contract event tables); (3) Do not build physical table until source is confirmed.
**Estimated Effort:** Small (renaming) + Medium (source identification)  
**Owner:** ETL + Business Analyst

---

### 4.11 Physical Tables With No Logical Entity (Orphaned Physical Tables)

#### `currency_units_conversion_rate` (91.9 MB, with CDF)

---

### Finding Orphaned-001 — currency_units_conversion_rate Belongs in silver.reference

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | `sa/subject-areas.md` §1 — Reference Data SA (`silver.reference`): "ISO Currencies, … FX Rates" |
| Evidence | Physical: `silver_dev_v2.account.currency_units_conversion_rate` (91.9 MB with Change Data Feed). SA taxonomy: FX rates → Reference SA |
| Affected Table | `silver.account.currency_units_conversion_rate` |
| Affected Column(s) | All |
| Confidence | 0.98 |

**Description:** FX conversion rates are reference data shared across all subject areas. Having them in `silver.account` means the Loan SA, Payment SA, and Treasury SA cannot use them without cross-SA joins in Silver (which is prohibited) or duplication. This is a critical integration failure.
**Remediation:** Move table to `silver.reference.currency_units_conversion_rate`. Update all Account SA pipelines to reference `silver.reference`. Govern as shared reference table.
**Estimated Effort:** Small (configuration change + one pipeline update)  
**Owner:** Data Engineering + Reference Data Steward

---

#### `current_bal_report` (0 bytes, empty)

### Finding Orphaned-002 — current_bal_report Is a Report/Aggregation Artifact

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | `SLV-007` — "No pre-computed metrics"; `NAM-003` — Silver entity naming: `<entity>` not `<entity>_report` |
| Evidence | Physical table named `current_bal_report` — "report" suffix indicates a pre-computed/aggregated artefact |
| Affected Table | `silver.account.current_bal_report` |
| Affected Column(s) | All |
| Confidence | 0.92 |

**Description:** A "balance report" is an aggregation, not a Silver-layer entity. Its presence (even empty) signals that someone is building Gold-layer outputs in Silver. The table has 0 bytes and no logical model entry.
**Remediation:** Drop this table from `silver.account`. If a balance report is needed, build it in `gold.retail.fact_account_daily_balance`.
**Estimated Effort:** Trivial (drop empty table + pipeline correction)  
**Owner:** Data Engineering

---

#### `pbd_account_master`, `ccod_account_master_table`, `sbca_master_table`

### Finding Orphaned-003 — Source-System-Prefixed Tables Fragment the Account Entity

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | High |
| Guideline Rule | `NAM-003` — Silver entity table: `<entity>` (no source-system prefix); `SLV-006` — 3NF (one canonical entity); guideline §2.2 Subject-Area Isolation |
| Evidence | Physical: `pbd_account_master` (PBD = pre-banking-day/product-specific prefix), `ccod_account_master_table` (CCOD = Credit Card OD), `sbca_master_table` (SBCA = Small Business CASA) — all empty (0 bytes) |
| Affected Table | `silver.account.pbd_account_master`, `silver.account.ccod_account_master_table`, `silver.account.sbca_master_table` |
| Affected Column(s) | All |
| Confidence | 0.94 |

**Description:** Three separate product-type-specific "account master" tables violate the canonical account entity principle. Silver should have one `account` entity with `account_type_code` discriminator, not separate tables per product type. Using source-system or product prefixes as table names violates NAM-003.
**Remediation:** Consolidate into a single canonical `silver.account.account` entity with `account_type_code` discriminator. If product-specific attributes are truly distinct, use companion tables (e.g., `silver.account.account_casa_attributes`, `silver.account.account_credit_card_attributes`). Drop the three empty prefixed tables.
**Estimated Effort:** Medium  
**Owner:** Data Modeling + ETL

---

#### `range_key_table` (0 bytes)

### Finding Orphaned-004 — range_key_table Is an ETL Artifact

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | Low |
| Guideline Rule | Governance — tables in production Silver must have a business purpose and logical model entry |
| Evidence | Physical: `silver_dev_v2.account.range_key_table` (0 bytes, no logical entity, cryptic name) |
| Affected Table | `silver.account.range_key_table` |
| Affected Column(s) | All |
| Confidence | 0.85 |

**Description:** `range_key_table` appears to be an ETL pipeline helper/lookup table (range partitioning key lookup) that has leaked into the Silver schema. It has no business purpose, no logical model entry, and 0 bytes.
**Remediation:** Investigate ETL pipeline that created this table. If it is an ETL internal artifact, remove it from the Silver schema and keep it in a pipeline-internal workspace schema. If it has a legitimate business purpose, create a logical model entry.
**Estimated Effort:** Small  
**Owner:** Data Engineering

---

### 4.12 Reference Entities in Account SA

Entities: `Reference_Account_Agreement_Type`, `Reference_Account_Type`, and the physical `reference_product`, `reference_currency_code`, `reference_financial_instrument_type`.

---

### Finding Reference-001 — Multiple Reference Tables in Wrong SA

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | Medium |
| Guideline Rule | `sa/subject-areas.md` §1 — Reference Data SA (`silver.reference`): "Master Lookups: ISO Currencies, Country Codes, … Industry Codes" |
| Evidence | Physical: `reference_currency_code`, `reference_financial_instrument_type`, `reference_product` all in `silver.account`. Logical: `Reference_Account_Agreement_Type`, `Reference_Account_Type` in Account SA |
| Affected Table | `silver.account.reference_currency_code`, `silver.account.reference_financial_instrument_type`, `silver.account.reference_product` |
| Affected Column(s) | All |
| Confidence | 0.93 |

**Description:** `reference_currency_code` and `reference_financial_instrument_type` are cross-cutting reference data and belong in `silver.reference`. `reference_product` belongs in `silver.product`. Having them in `silver.account` forces every other SA to either duplicate data or make prohibited cross-SA joins. `Reference_Account_Agreement_Type` and `Reference_Account_Type` are account-specific lookups and may stay in `silver.account` as they are not cross-cutting.
**Remediation:** Move `reference_currency_code` → `silver.reference.currency_code`; `reference_financial_instrument_type` → `silver.reference.financial_instrument_type` or `silver.treasury.financial_instrument_type`; `reference_product` → `silver.product.product`. Retain account-specific reference tables in `silver.account`.
**Estimated Effort:** Small  
**Owner:** Data Engineering + Reference Data Steward

---

### 4.13 Credit_Card_Account Entity (In Mapping, Not in Logical Model)

---

### Finding CreditCard-001 — Credit Card Account Entity Mapped in Account SA Workbook But Belongs in Card SA

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | `sa/subject-areas.md` §7 — Card SA (`silver.card`): "Plastic & Revolving Details: Card Numbers, Credit Limits, Billing Cycles…" |
| Evidence | `account_data_mapping.xlsx` Detail3 sheet contains `Credit_Card_Account` entity sourced from CAPS system with attributes: account number, card type, open date, application number, total payments since cut-off, processing day — not present in Account logical model |
| Affected Table | `silver.account.ccod_account_master_table` (likely physical counterpart) |
| Affected Column(s) | Entire entity |
| Confidence | 0.88 |

**Description:** Credit card account data belongs in `silver.card`. Mapping and potentially loading it into `silver.account` (via `ccod_account_master_table`) will fragment card data between two schemas and break the Card SA completeness. "Total payments since cut-off" is also an aggregation metric (violates SLV-007).
**Remediation:** Move `Credit_Card_Account` to Card SA logical model and physical schema. Update `ccod_account_master_table` to be a `silver.card.credit_card_account` table (or consolidated into `silver.card.card`).
**Estimated Effort:** Medium  
**Owner:** Data Modeling + Card SA Owner

---

### 4.14 Missing Critical Canonical Entity: `account`

---

### Finding MissingEntity-001 — No Canonical account Entity in Logical Model or Physical Schema

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | `sa/subject-areas.md` §5 — Account SA description: "Unified Balance records for CASA, Term Deposits, Loan Accounts, and Card Balances. Tracks lifecycle status." BIAN Account service domain |
| Evidence | Logical model Account SA: 30 entities, none named simply `account` or `account_master` in the IFW sense. Physical: no `account` table. The closest is `account_agreement_master` which is over-fat and conflates agreement with account |
| Affected Table | Missing: `silver.account.account` |
| Affected Column(s) | All |
| Confidence | 0.99 |

**Description:** The core of the Account SA — a lean `account` entity (account_sk, account_bk, account_type_code, account_status_code, open_date, close_date, currency_code, party_bk, product_bk, branch_code, + SCD-2 + metadata) — does not exist. Every downstream consumer (Gold dimensions, IFRS 9 staging, CBUAE balance reporting) needs this central entity. Without it, the Account SA cannot serve its primary purpose.
**Remediation:** Create `silver.account.account` as the canonical holding container entity. Decompose `Account_Agreement_Master` into: (1) `account` (master/identity), (2) `account_balance` (balance amounts), (3) `account_party_role` (joint holders). Reference the physical `account_eod_balance` as the balance snapshot table and ensure it is linked to `account.account_bk`.
**Estimated Effort:** Large  
**Owner:** Data Modeling + ETL

---

## 5. Denormalization Register

| # | Denormalization | Entity | Type | Justified? | Notes |
|---|---|---|---|---|---|
| DN-001 | Product description in account master | `Account_Agreement_Master` | Unnecessary | ❌ No | Violates 3NF/SA isolation; must be removed; join in Gold |
| DN-002 | Party segment in account master | `Account_Agreement_Master` | Unnecessary | ❌ No | Party attribute; create update anomaly; remove and join in Gold |
| DN-003 | Joint holder names as flat columns | `Account_Agreement_Master` | Unnecessary | ❌ No | Repeating group; must normalize to `account_party_role` child table |
| DN-004 | MTD balance metrics in account master | `Account_Agreement_Master` | Unnecessary | ❌ No | Aggregation; must be removed from Silver entirely |
| DN-005 | IBAN/SWIFT in account master | `Account_Agreement_Master` | Acceptable | ⚠️ Marginal | IBAN is an account-identifying attribute; acceptable if kept as single IBAN; SWIFT/routing should move to Payment SA |
| DN-006 | Collateral data in Account SA | Exposure entities | Unnecessary | ❌ No | Wrong SA entirely; relocate to silver.collateral |
| DN-007 | Risk grade/metrics in Account SA | Risk entities | Unnecessary | ❌ No | Wrong SA entirely; relocate to silver.risk |
| DN-008 | Currency/FX data in Account SA | Physical tables | Unnecessary | ❌ No | Reference data SA; relocate |
| DN-009 | Balance currency code in balance summary | `Account_Balance_Summary_DD` | Necessary | ✅ Yes | Currency code alongside balance amount is correct normalization for multi-currency |
| DN-010 | Account type/status codes stored raw (from source) | Multiple | Unnecessary | ❌ No | Raw source codes must be resolved via `silver.reference.code_mapping` per SLV-004 |

**Summary:** 9 of 10 identified denormalizations are unjustified (Unnecessary or Wrong SA). Only 1 is appropriate. This represents a widespread 3NF and SA isolation gap throughout the model.

---

## 6. Guideline Compliance Summary

| Rule | Description | Status | Entities Affected |
|---|---|---|---|
| **SLV-001** | `<entity>_sk` (MD5) on every entity | 🔴 **Fail** — no entity has `*_sk` | All 30 logical entities |
| **SLV-002** | `<entity>_bk` on every entity | 🔴 **Fail** — natural keys used as apparent PK | All 30 logical entities |
| **SLV-003** | SCD-2 default | 🟡 **Partial** — `account_agreement_master` has CDF; others unclear; SCD columns wrongly typed (CHAR18) | Most entities |
| **SLV-004** | No source codes; use code_mapping | 🔴 **Fail** — raw source codes visible in multiple mappings (ACCOUNTTYPE raw, SCHM_TYPE raw) | `account_agreement_master`, `account_balance_summary_dd` |
| **SLV-005** | DQ-gated writes; quarantine table | 🟡 **Partial** — `account_agreement_master` has `checkConstraints` feature; no `data_quality_status` column in logical model | All entities |
| **SLV-006** | 3NF; no derived values | 🔴 **Fail** — multiple 3NF violations (joint holders, product desc, party attrs) | `account_agreement_master`, multiple |
| **SLV-007** | No pre-computed metrics | 🔴 **Fail** — MTD amounts, MTD balance, balance count are aggregations | `account_agreement_master`, `account_balance_summary_dd` |
| **SLV-008** | All metadata columns present | 🔴 **Fail** — 6+ required columns missing; existing SCD columns wrongly typed | All entities |
| **SLV-009** | All timestamps UTC | 🟡 **Unknown** — cannot confirm without DDL; source `DATE` types need UTC conversion at ingestion | All entities |
| **SLV-010** | Monetary amounts in fils (AED smallest unit) | 🔴 **Fail** — all amounts typed STRING; no evidence of fils conversion | All monetary attributes |
| **NAM-001** | snake_case, lowercase | 🟡 **Partial** — physical tables are lowercase; logical model uses PascalCase (correct for logical); CHAR(18) data types are Erwin artifacts | Physical tables mostly compliant |
| **NAM-003** | Table naming: `<entity>` for Silver | 🔴 **Fail** — `pbd_account_master`, `ccod_account_master_table`, `sbca_master_table`, `current_bal_report`, `range_key_table` all violate | 5 physical tables |
| **NAM-005** | No SQL reserved words | ✅ **Pass** — no obvious reserved word violations in visible column names | — |
| **DQ-001** | Error rate < 5% triggers fail + alert | 🟡 **Unknown** — cannot verify pipeline configuration | All pipelines |
| **PII** | EID, card number, phone, passport SHA-256 masked | 🔴 **Fail** — `primary_account_holder` stores party name unmasked; joint holder names unmasked | `account_agreement_master` |

**Compliance Score: 2 / 15 rules fully compliant (NAM-001 partial, NAM-005). 9 rules fail. 4 unknown/partial.**

---

## 7. Remediation Plan

### Phase 1 — Immediate (P0, Sprint 1–2)

| # | Action | Entity / Table | Owner | Effort | Guideline |
|---|---|---|---|---|---|
| 1 | Create canonical `silver.account.account` entity with `account_sk` (MD5), `account_bk`, `account_type_code`, `account_status_code`, `open_date`, `close_date`, `currency_code`, `party_bk` (FK), `product_bk` (FK), `branch_bk` (FK), + SCD-2 columns | New entity | Data Modeling | Large | SLV-001/002/003 |
| 2 | Register `account_eod_balance` in Erwin logical model; add all mandatory metadata columns; register in GDGC | `account_eod_balance` | Data Modeling + Governance | Medium | §2.11, SLV-008 |
| 3 | Relocate collateral/exposure entities to `silver.collateral` | 8 entities | Data Modeling | Small | SA taxonomy |
| 4 | Relocate risk entities to `silver.risk` | 5 entities | Data Modeling | Small | SA taxonomy |
| 5 | Relocate `Agreement` entity to `silver.contract` | 1 entity | Data Modeling | Small | SA taxonomy |
| 6 | Move `currency_units_conversion_rate` + `reference_currency_code` to `silver.reference` | 2 tables | Data Engineering | Small | SA taxonomy |
| 7 | Move `financial_instrument` + `reference_financial_instrument_type` to `silver.treasury` | 2 tables | Data Engineering | Small | SA taxonomy |
| 8 | Move `reference_product` to `silver.product` | 1 table | Data Engineering | Small | SA taxonomy |
| 9 | Fix 4 confirmed semantic mapping errors (contract name, contract expiry, segment, principal vs available balance) | `account_agreement_master` | ETL | Medium | Mapping accuracy |
| 10 | Drop `current_bal_report` (empty, aggregation artifact) | 1 table | Data Engineering | Trivial | SLV-007 |
| 11 | Mask party name in `primary_account_holder`, `1st_joint_account_holder`, `2nd_joint_account_holder` with SHA-256 | `account_agreement_master` | ETL | Small | DQ Controls PII §10 |
| 12 | Relocate `Credit_Card_Account` mapping to Card SA | Mapping workbook | Data Modeling | Small | SA taxonomy |

### Phase 2 — Next Sprint (P1, Sprint 3–5)

| # | Action | Entity / Table | Owner | Effort |
|---|---|---|---|---|
| 13 | Add `account_sk` (MD5) + `account_bk` to `account_agreement_master`; add all mandatory SCD-2 and metadata columns with correct types | `account_agreement_master` | Data Modeling + ETL | Large |
| 14 | Fix all attribute data types (STRING → DATE/TIMESTAMP/DECIMAL/BOOLEAN/BIGINT) across all entities | All | Data Modeling | Large |
| 15 | Remove MTD aggregation columns from `account_agreement_master` and `account_balance_summary_dd`; build Gold `fact_account_monthly` | Multiple | Data Modeling + ETL | Medium |
| 16 | Decompose joint holders into `account_party_role` child table; remove flat columns from master | `account_agreement_master` | Data Modeling | Medium |
| 17 | Remove product description and party segment from `account_agreement_master` | `account_agreement_master` | Data Modeling | Small |
| 18 | Build physical tables for `account_group`, `account_account_group`, `account_group_limit_hist` | 3 entities | ETL | Medium |
| 19 | Fix `account_balance_summary_dd` semantic error (currency code column named as amount) | `account_balance_summary_dd` | ETL | Small |
| 20 | Rename `Account_Product_Histroy` → `Account_Product_History`; rename `Account_Renewal_Histroy` → `Account_Renewal_History` in Erwin | 2 entities | Data Modeling | Trivial |
| 21 | Consolidate `pbd_account_master`, `ccod_account_master_table`, `sbca_master_table` into canonical `account` entity with `account_type_code` discriminator | 3 tables | ETL | Medium |
| 22 | Remove `range_key_table` from Silver schema | 1 table | Data Engineering | Trivial |
| 23 | Implement code mapping via `silver.reference.code_mapping` for `account_type_code`, `account_status_code`, `balance_category_type_code` | Multiple | ETL | Medium |

### Phase 3 — Backlog (P2, Sprint 6+)

| # | Action | Owner | Effort |
|---|---|---|---|
| 24 | Source identification for empty-table mappings: joint holders, account maturity date, hold codes, Account_Renewal_History | Business Analyst + ETL | Medium |
| 25 | Implement DQ rules + quarantine pattern for `account_agreement_master` | Data Engineering | Medium |
| 26 | Add GDGC catalog registration for all Account SA tables | Governance | Small |
| 27 | UTC timestamp enforcement audit across all Account SA pipelines | Data Engineering | Small |
| 28 | Resolve 15 "Pending for Modelers" + 8 "Pending for RAK Confirmation" attributes | Data Modeling + Business | Medium |
| 29 | Build `account_identifier` child entity for IBAN/SWIFT/routing | Data Modeling | Small |
| 30 | Create `account_status_history` entity for full lifecycle tracking | Data Modeling | Medium |

### Indicative Schedule

| Phase | Sprint | Key Deliverable |
|---|---|---|
| P0 Foundation | S1 | SA relocations complete; canonical `account` entity in Erwin; `account_eod_balance` registered |
| P0 Critical | S2 | Mapping errors fixed; PII masking deployed; stray tables removed |
| P1 Core | S3–S4 | `account_agreement_master` restructured with SK/BK/SCD-2; data types corrected |
| P1 Build | S5 | Account group tables built; balance entity corrected |
| P2 Polish | S6+ | Unmapped attributes sourced; DQ controls complete; catalog registration |

---

## 8. Appendix

### Appendix A: Mapping Summary by Entity

| Entity | Total Attrs | Mapped | Not Available | Blank/Pending |
|---|---|---|---|---|
| Account_AGREEMENT_Master | 35 | 28 | 7 | 0 |
| Account_Balance_Summary_DD | 11 | 9 | 2 | 0 |
| Account_Group | 8 | 7 | 1 | 0 |
| Account_Group_Criterion | 2 | 1 | 1 | 0 |
| Account_Renewal_Histroy | 5 | 3 | 0 | 2 (empty tables) |
| Account_Account_Group | 6 | 5 | 1 | 0 |
| Agreement | 22 | 10 | 7 | 5 |
| Account_Product_Histroy | 6 | 6 | 0 | 0 |
| Account_Group_Risk_Hist | 13 | 4 | 4 | 5 |
| Account_Group_Risk_Type_Hist | 9 | 5 | 1 | 3 (mismatched) |
| Account_Group_Score_DD | 7 | 2 | 2 | 3 |
| Account_Group_Bal_Hist_DD | 9 | 6 | 1 | 2 |
| Account_Group_Related | 8 | 4 | 1 | 3 |
| Account_Group_Type_Assoc | 7 | 4 | 0 | 3 |
| Account_Group_Limit_Hist | 9 | 4 | 2 | 3 |
| Account_Group_Risk_Grade | 9 | 2 | 4 | 3 |
| Account_Access_Device | 6 | 4 | 2 | 0 |
| Account_Access_Device_Feature | 8 | 7 | 1 | 0 |
| Reference_* (14 ref entities) | ~100 | ~65 | ~20 | ~15 |

**Total:** 342 logical attributes. Approximately 81% have some mapping; ~6.4% confirmed Not Available; ~12.6% pending/blank.

### Appendix B: Guideline Citations

| Rule Code | Source Document | Section | Key Requirement |
|---|---|---|---|
| SLV-001/002 | 11-modeling-dos-and-donts.md | §2.4 | MD5 surrogate key + business key on every Silver entity |
| SLV-003 | 11-modeling-dos-and-donts.md | §2.3 | SCD Type 2 default; `effective_from`, `effective_to`, `is_current` |
| SLV-004 | 11-modeling-dos-and-donts.md | §2.5 | Canonical code mapping via `silver.reference.code_mapping` |
| SLV-005 | 01-data-quality-controls.md | §3.2 | DQ-gated writes; quarantine on failure |
| SLV-006 | 11-modeling-dos-and-donts.md | §2.1 | 3NF; no repeating groups; decompose repeating groups to child tables |
| SLV-007 | 11-modeling-dos-and-donts.md | §2.6 | No aggregations or derived metrics in Silver |
| SLV-008 | 11-modeling-dos-and-donts.md | §2.7 | All 11 mandatory audit columns with canonical names |
| SLV-009 | 11-modeling-dos-and-donts.md | §2.9 | All timestamps in UTC |
| SLV-010 | 01-data-quality-controls.md | §3.1 | Monetary amounts `DECIMAL(18,4)`; AED in fils |
| NAM-001 | 06-naming-conventions.md | General Rules | snake_case, lowercase, no SQL reserved words |
| NAM-003 | 06-naming-conventions.md | Table Naming | Silver entity: `<entity>` (no prefix/suffix) |
| PII | 01-data-quality-controls.md | §10 | SHA-256 masking for EID, card, phone, passport, names |
| §2.2 SA Isolation | 11-modeling-dos-and-donts.md | §2.2 | No cross-SA joins in Silver pipelines |
| §2.11 Erwin-first | 11-modeling-dos-and-donts.md | §2.11 | Logical model approved before DDL |
| §2.12 Catalog | 11-modeling-dos-and-donts.md | §2.12 | All Silver tables registered in GDGC |

### Appendix C: Industry References

| Standard | Relevance to Account SA |
|---|---|
| **BIAN v11 — Current Account** | Account master (account_sk, type, status, open/close dates), Account Balance (daily snapshot), Account Feature — canonical separation of account identity from balance |
| **IFW (IBM Banking Data Warehouse)** | `ACCOUNT` central entity; `ACCOUNT_BALANCE` as child (one per balance type per date); `ACCOUNT_ARRANGEMENT` for product terms — analogous to the decomposition recommended in this report |
| **BCBS 239** | Principle 4 (Accuracy): FX rates must be shared reference; Principle 6 (Completeness): all account types in one SA; balance reporting must be traceable to atomic transactions |
| **IFRS 9** | ECL staging requires: account balance, maturity date (EAD), credit quality (PD/LGD) — maturity date is currently "Not Available", PD/LGD incorrectly placed in Account SA |
| **CBUAE Regulatory Reporting** | Requires unified CASA + Term Deposit + Loan + Card balance view per customer — requires canonical `account` entity with `account_type_code` discriminator and `account_eod_balance` as source of truth |
| **PDPL (UAE Personal Data Protection Law) Art. 5** | Consent check before Gold promotion; SHA-256 masking for PII in Silver — party names in account entity violate this |

---

**Report ends. Total findings: 20 named findings + 5 section-level findings. P0: 13 findings. P1: 5 findings. P2: 2 findings.**