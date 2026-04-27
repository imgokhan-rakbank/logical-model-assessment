# Gap Assessment Report — Collateral

| Field | Value |
|---|---|
| Subject Area | `collateral` |
| Schema | `silver.collateral` |
| Assessment Date | 2025-07-15 |
| Overall State | 🔴 Incorrect |
| Entities Assessed | 12 |
| Total Findings | 47 (P0: 18 · P1: 19 · P2: 10) |
| Confidence | 0.82 |

---

## 1. Executive Summary

### 1A. Consolidated Executive Summary (Management Presentation)

**Overall Health: 🔴 Incorrect** — The Collateral subject area contains 12 logical entities, none of which fully comply with platform silver-layer standards. The most critical risk is the **complete absence of surrogate keys (`<entity>_sk`) and canonical business keys (`<entity>_bk`)** on every entity, which breaks downstream Gold-layer identity linkage and IFRS 9 collateral-coverage reporting.

**Top Risks:**
- No surrogate or business keys on any of the 12 entities (SLV-001/SLV-002 violation; P0)
- All logical data types are `CHAR(18)` — monetary, date, and flag fields require correct types (P0)
- 7 non-production tables (backup/test/dummy) polluting the production schema (P0)
- 2 transaction tables from a different subject area incorrectly placed in `silver.collateral` (P0)
- Mandatory silver audit columns partially absent or non-standardly named on all entities (P1)

### Top 5 Actions

| # | Action | Priority | Owner |
|---|---|---|---|
| 1 | Add `<entity>_sk` (MD5 surrogate) + `<entity>_bk` (business key) columns to all 12 entities | P0 | Data Modeling / ETL |
| 2 | Drop/move the 7 non-production tables (backup, test, dummy) and 2 mis-scoped transaction tables from `silver.collateral` | P0 | DBA / Data Engineering |
| 3 | Correct all CHAR(18) logical data types: monetary amounts to DECIMAL(18,4), dates to DATE, timestamps to TIMESTAMP, flags to BOOLEAN | P0 | Data Modeling |
| 4 | Standardize and complete all 11 mandatory silver audit columns on every entity table | P1 | ETL / Data Modeling |
| 5 | Resolve 26 unmapped attributes across sub-asset entities (especially Precious Metal, Inventory, Securities, Crypto) | P1 | Business Analysis / ETL |

**KPIs to track progress:**
- % entities with `_sk` + `_bk` implemented: **0 / 12 = 0%** (target: 100%)
- % attributes with confirmed mapped status: approx. **54 / 122 = 44%** (target: ≥ 90%)

**Recommended management decision:** Approve a dedicated sprint (2 weeks) to remediate P0 structural issues before connecting Collateral Silver to any Gold or regulatory reporting pipeline. Current state precludes reliable IFRS 9 collateral-coverage computation.

---

**Technical Executive Summary:**

The Collateral subject area (`silver.collateral`) is scoped as sub-area 8b under the Lending hierarchy and covers pledged security assets (Real Estate, Metals, Cash/Fixed Deposits, Securities, Vehicles, Guarantees, Receivables, Crypto, Intellectual Property, Inventory) and the loan-collateral pledge association table. Twelve logical entities are defined in the bank's logical model. Physical Delta tables exist for 10 of the 12 entities; `Customer_Collateral_IP` and `Customer_Collateral_Inventory` have no corresponding physical table. The schema additionally contains 7 non-production artefacts (backup, test, dummy, and WBG-variant tables) and 2 completely mis-scoped tables (`non_electronic_channel_transaction__phonebanking_` and `non_electronic_channel_transaction__atm_`) that belong to `silver.transaction`, not to collateral. Every entity violates SLV-001/SLV-002 (no surrogate or business keys), the entire logical model uses `CHAR(18)` as a catch-all data type (wrong for amounts, dates, booleans, and codes), mandatory silver audit column sets are either absent or non-standardly named, and 26 attributes across the sub-asset entities remain unmapped. Two critical domain entities — **Collateral Valuation History** and **Lien/Charge Registration** — are entirely missing from the model, preventing IFRS 9 ECL collateral-coverage calculations. The subject area is rated 🔴 **Incorrect** with confidence 0.82.

---

## 2. Subject Area Architecture Assessment

### 2.1 Domain Scoping & BIAN Alignment

**Finding:** ✅ Scoping is correct in principle but has execution problems.

The `collateral` subject area is correctly identified in `sa/subject-areas.md` as sub-area **8b** within the Lending hierarchy (`silver.collateral`), with schema `silver.collateral` and priority 2 (High). The description — *"Security pledged against lending: Real Estate, Metals, Cash, Securities. Asset Valuations, LTV Ratios, and Lien/Charge Registrations"* — aligns with the **BIAN v11 "Collateral Asset Administration"** service domain. The entity set covers the correct asset class taxonomy.

**Issues:**
1. Two tables in `silver.collateral` — `non_electronic_channel_transaction__phonebanking_` and `non_electronic_channel_transaction__atm_` — belong to `silver.transaction`, not to this subject area. Both have zero rows but exist as registered Delta tables in the catalog. This is a schema contamination / mis-deployment error.
2. The entity prefix `Customer_` on every collateral entity (e.g., `Customer_CollateralAsset`, `Customer_Collateral_Real_Estate`) implies a customer-centric design rather than a clean collateral-asset-centric design. Under BIAN, collateral assets have an independent lifecycle (they can be re-pledged across facilities or held without an active loan). The customer link should be expressed via FK to the Party/Customer entity, not embedded in the entity name.
3. **Missing:** The subject area description mentions LTV Ratios and Lien/Charge Registrations, but no dedicated `collateral_valuation` (history) or `collateral_lien_registration` entity exists in the logical model.

### 2.2 Identity Strategy

**Finding:** 🔴 No surrogate keys or canonical business keys on any entity.

The logical model defines `Collateral id` as the PK for `Customer_CollateralAsset` and sub-type entities. This is a source-system ID from RLS/FIN_LEAM, not a platform surrogate key. No `<entity>_sk` (MD5 deterministic surrogate) or `<entity>_bk` (canonical business key) column exists in any entity.

| Requirement | Status |
|---|---|
| `<entity>_sk` (MD5 surrogate) on every entity | ❌ Absent on all 12 entities |
| `<entity>_bk` (canonical business key) on every entity | ❌ Absent on all 12 entities |
| Deterministic MD5 derivation | ❌ Not applicable — SKs not defined |
| Cross-entity FK integrity (`collateral_sk` in pledge table) | ❌ No SKs defined; FK cannot be implemented |

Downstream impact: Gold-layer fact tables cannot join to Collateral dimension tables. IFRS 9 ECL collateral-coverage reporting will be broken.

### 2.3 SCD Strategy

**Finding:** 🟡 Partial — Non-standard SCD columns present in logical model; standard columns missing.

The logical model for sub-asset entities (e.g., `Customer_Collateral_Vehicles`, `Customer_Collateral_Real_Estate`) includes `Business Start Date`, `Business End Date`, and `Is Active Flag`. These are a custom SCD-2 approximation but do **not** match the required column names from guideline §2.7:

| Required Column | Present in LM | Correct Name? |
|---|---|---|
| `effective_from` (TIMESTAMP) | 🟡 `Business Start Date` | ❌ Wrong name |
| `effective_to` (TIMESTAMP) | 🟡 `Business End Date` | ❌ Wrong name |
| `is_current` (BOOLEAN) | 🟡 `Is Active Flag` | ❌ Wrong name; also STRING vs BOOLEAN |
| `source_system_code` | ⚠️ Only in CollateralAsset | ❌ Missing from all sub-entities |
| `source_system_id` | ⚠️ Only in CollateralAsset | ❌ Missing from all sub-entities |
| `create_date` | ⚠️ Only in CollateralAsset | ❌ Missing from all sub-entities |
| `update_date` | ⚠️ Only in CollateralAsset | ❌ Missing from all sub-entities |
| `delete_date` | ⚠️ Only in CollateralAsset | ❌ Missing from all sub-entities |
| `run_id` | ❌ Absent | ❌ Missing from all entities |
| `source_ingestion_date` | ❌ Absent | ❌ Missing from all entities |

`Customer_CollateralAsset` has the most complete metadata but uses `IsLogicallyDeleted` (non-standard name and type), `Deleted date` (non-standard), and `Create date`/`Update date` (missing type precision). None of the entities follow the standard SCD-2 pattern from guideline §2.3.

### 2.4 Cross-Entity Relationships & Cardinality

**Finding:** 🔴 No FK relationships enforced or documented in logical model.

Expected FK relationships that are absent:

| Relationship | Expected FK | Status |
|---|---|---|
| `Customer_CollateralAsset` → Party (`customer_sk`) | `customer_sk` | ❌ Only `Customer number (CIF)` (source BK, no SK) |
| `Customer_Collateral_[subtype]` → `Customer_CollateralAsset` (`collateral_sk`) | `collateral_sk` | ❌ Only raw `Collateral ID` (source ID, no SK) |
| `Customer_Loan_Collateral_Pledges` → `Customer_CollateralAsset` | `collateral_sk` | ❌ Only `Collateral ID` (source BK) |
| `Customer_Loan_Collateral_Pledges` → Loan (`loan_sk`) | `loan_sk` or `loan_account_bk` | ❌ Only `Loan Account number` (string, no SK) |
| `Customer_Collateral_Guarantees` → Party (guarantor) | `party_sk` | ❌ Only `Guarantor name` (denormalized string) |

The supertype-subtype relationship between `Customer_CollateralAsset` (the abstract asset header) and the individual sub-asset entities (Real Estate, Vehicles, etc.) is implied but no FK is defined. Cardinality (1:0..1 between header and sub-type) is not enforced.

### 2.5 ETL Mapping & Lineage Completeness

**Finding:** 🟡 Partial coverage; significant unmapped attributes and two missing physical tables.

| Entity | Total Attrs (LM) | Mapped Attrs | Unmapped/NA | Physical Table |
|---|---|---|---|---|
| Customer_CollateralAsset | 29 | 16 | 10 | ✅ customer_collateralasset |
| Customer_Collateral_Real_Estate | 9 | 6 | 0 (3 metadata) | ✅ customer_collateral_real_estate |
| Customer_Collateral_Vehicles | 8 | 4 | 1 (VIN) | ✅ customer_collateral_vehicles |
| Customer_Collateral_Fixed_Deposit | 8 | 4 | 1 (Haircut) | ✅ customer_collateral_fixed_deposit |
| Customer_Collateral_Precious_Metal | 8 | 1 | 4 | ✅ customer_collateral_precious_metal |
| Customer_Collateral_Securities | 8 | 2 | 3 | ✅ customer_collateral_securities |
| Customer_Collateral_Guarantees | 8 | 3 | 2 | ✅ customer_collateral_guarantees |
| Customer_Loan_Collateral_Pledges | 10 | 7 | 0 | ✅ customer_loan_collateral_pledges |
| Customer_Collateral_Receivables | 10 | 5 | 2 | ✅ customer_collateral_receivables |
| Customer_Collateral_Crypto | 8 | 3 | 2 | ✅ customer_collateral_crypto |
| Customer_Collateral_IP | 8 | 4 | 1 | ❌ **No physical table** |
| Customer_Collateral_Inventory | 8 | 1 | 4 | ❌ **No physical table** |

Source systems: **RLS** (Retail Lending System) via schemas `FIN_LEAM` (Finacle Leasing/Asset Management) and `LOS0909` (Loan Origination System). All mappings use a single source system; no multi-source companion table pattern required. Lineage documented at attribute level in `Customer_Collateral_D_Model` sheet.

### 2.6 Major Design Gaps

| Gap | Category | Priority | Confidence |
|---|---|---|---|
| No `<entity>_sk` + `<entity>_bk` on any of 12 entities | Identity | P0 | 0.95 |
| All logical types are CHAR(18) — wrong for all non-string fields | Data Type | P0 | 0.97 |
| 2 transaction tables mis-scoped into collateral schema | Architecture | P0 | 0.99 |
| 7 non-production tables (backup/test/dummy) in production schema | Governance | P0 | 0.99 |
| Missing `collateral_valuation` entity for IFRS 9 LTV history | Architecture | P0 | 0.90 |
| Missing `collateral_lien_registration` entity | Architecture | P1 | 0.85 |
| `Customer_Collateral_IP` and `Customer_Collateral_Inventory` have no physical tables | Lineage | P1 | 0.99 |
| Non-standard / incomplete silver audit column set on all entities | SCD/Metadata | P1 | 0.97 |
| Monetary amounts typed as STRING in data mapping | Data Type | P0 | 0.95 |
| No FK enforcement or documentation between header and sub-type entities | Lineage | P1 | 0.93 |
| Denormalized party attributes (guarantor name, lien holder, debtor name) | 3NF | P1 | 0.90 |
| Raw source codes passed through without `silver.reference.code_mapping` | SLV-004 | P1 | 0.85 |

---

## 3. Entity Inventory

| Entity | State | Criticality | Mapped Tables | Unmapped Attrs | P0 | P1 | P2 | Confidence |
|---|---|---|---|---|---|---|---|---|
| Customer_CollateralAsset | 🔴 Incorrect | High | customer_collateralasset | 10 | 3 | 4 | 2 | 0.88 |
| Customer_Collateral_Real_Estate | 🔴 Incorrect | High | customer_collateral_real_estate | 0 | 2 | 3 | 1 | 0.85 |
| Customer_Collateral_Vehicles | 🔴 Incorrect | Medium | customer_collateral_vehicles | 1 | 2 | 3 | 1 | 0.83 |
| Customer_Collateral_Fixed_Deposit | 🔴 Incorrect | Medium | customer_collateral_fixed_deposit | 1 | 2 | 3 | 1 | 0.83 |
| Customer_Collateral_Precious_Metal | 🔴 Incorrect | Medium | customer_collateral_precious_metal | 4 | 2 | 3 | 1 | 0.80 |
| Customer_Collateral_Securities | 🔴 Incorrect | High | customer_collateral_securities | 3 | 2 | 3 | 1 | 0.82 |
| Customer_Collateral_Guarantees | 🔴 Incorrect | High | customer_collateral_guarantees | 2 | 2 | 3 | 2 | 0.82 |
| Customer_Loan_Collateral_Pledges | 🔴 Incorrect | High | customer_loan_collateral_pledges | 0 | 2 | 3 | 1 | 0.88 |
| Customer_Collateral_Receivables | 🔴 Incorrect | High | customer_collateral_receivables | 2 | 2 | 3 | 1 | 0.85 |
| Customer_Collateral_Crypto | 🔴 Incorrect | Low | customer_collateral_crypto | 2 | 2 | 2 | 1 | 0.75 |
| Customer_Collateral_IP | 🔴 Incorrect | Low | **None** | 5 | 1 | 3 | 1 | 0.78 |
| Customer_Collateral_Inventory | 🔴 Incorrect | Low | **None** | 4 | 1 | 3 | 1 | 0.78 |

---

## 4. Per-Entity Assessments

---

### 4.1 `Customer_CollateralAsset`

**State:** 🔴 Incorrect  
**Criticality:** High — This is the master collateral asset header entity that every sub-type and pledge entity references. Data quality failures here cascade to all downstream uses including IFRS 9 collateral-coverage and CBUAE collateral reporting.  
**Confidence:** 0.88  
**Physical Table(s):** `silver.collateral.customer_collateralasset`

#### Industry Fitness

`Customer_CollateralAsset` is designed as a **collateral asset supertype** — a header entity that holds common attributes (collateral type, value, haircut, lien status) shared by all collateral sub-types. This pattern is sound and aligns with the BIAN "Collateral Asset Administration" service domain. However, several concerns arise:

1. **Mixed responsibility:** `Customer number (CIF)` directly embedded in the asset header couples collateral identity to a specific customer. In reality, collateral assets can be shared (pledged by multiple borrowers) or transferred. The customer linkage should be expressed via FK to the Party entity, not embedded as a column.
2. **Liquidity score** is a derived/computed metric (mapped from `LOS_APP_APPLICATIONS.LAA_APP_CRSCORE_N` which is a credit score, not a liquidity score). Pre-computed scores should not be stored in a Silver entity (SLV-007 violation). Additionally, using a credit score as a liquidity score proxy is semantically incorrect.
3. **Missing haircut decomposition:** `Base haircut`, `Risk adjustment`, and `Effective haircut` are modeled as separate attributes but all are unmapped. This haircut breakdown is required for BCBS 239 / CRR Article 194 collateral eligibility calculations.
4. **No valuation history support:** `Current value` and `Valuation date` are point-in-time attributes on the header entity. IFRS 9 requires LTV to be computed from a series of valuations over time, which requires a separate `collateral_valuation` child entity.
5. **Grain concern:** The entity grain is one row per collateral asset (identified by `Collateral id`). The presence of `Business Start Date` / `Business End Date` suggests SCD-2 is intended but the implementation is non-standard.

#### Attribute Review

| Logical Attribute | Mapping Status | Physical Column | Actual Type | Expected Type | Nullable (Actual) | Nullable (Expected) | Key Role | SCD Required | Issues |
|---|---|---|---|---|---|---|---|---|---|
| `Collateral id` | Mapped | `COLLATERALID` (LEA_COLLATERAL_DTL) | NUMBER(8) | STRING (BK) | NO | NO | BK / PK | SCD-2 | ❌ Source ID used as PK; no `collateral_sk` surrogate defined |
| `Customer number (CIF)` | Mapped | `CIF_NO` (NBFC_COUNTRY_M) | VARCHAR | STRING | YES | NO | FK | - | ❌ Denormalized party attr; should be FK `party_bk` to silver.party |
| `Owner group id` | Unmapped | — | — | STRING | — | YES | FK | - | 🟡 No source identified; business concept unclear |
| `Collateral type` | Mapped | `LAC_COLLATERAL_TYPE_C` | CHAR | STRING (code) | YES | NO | — | - | ❌ Raw source code; must map via `silver.reference.code_mapping` |
| `Collateral source` | Unmapped | — | — | STRING | — | YES | — | - | 🟡 No source identified |
| `Original value` | Mapped | `LAC_COLLATERAL_VALUE_N` | NUMBER | DECIMAL(18,4) | YES | NO | — | SCD-2 | ❌ Typed STRING in mapping; must be DECIMAL(18,4); missing `currency_code` |
| `Current value` | Mapped | `N_MARKET_VALUE` (LEA_PROPERTY_DTL) | NUMBER | DECIMAL(18,4) | YES | NO | — | SCD-2 | ❌ Point-in-time only; no valuation history; typed STRING; must be DECIMAL(18,4) |
| `Base haircut` | Unmapped | — | — | DECIMAL(5,4) | — | NO | — | - | ❌ Unmapped; critical for BCBS 239 collateral eligibility |
| `Risk adjustment` | Unmapped | — | — | DECIMAL(5,4) | — | NO | — | - | ❌ Unmapped; required for regulatory LTV |
| `Effective haircut` | Unmapped | — | — | DECIMAL(5,4) | — | NO | — | - | ❌ Unmapped; = Base haircut + Risk adjustment |
| `Valuation date` | Mapped | `APPRAISAL_DATE` (LEA_PROPERTY_DTL) | DATE | DATE | YES | NO | — | SCD-2 | ⚠️ Typed CHAR(18) in LM; must be DATE |
| `Is Pledged` | Unmapped | — | — | BOOLEAN | — | YES | — | - | ❌ Unmapped; important status indicator |
| `Is Active` | Unmapped (partial) | — | — | BOOLEAN | — | NO | — | - | ❌ Source listed as `RLS.None.None.None`; effectively unmapped |
| `Lien priority` | Mapped | `LAC_LIEN_TITLE_FLAG_C` | CHAR | STRING (code) | YES | YES | — | - | ⚠️ Raw source code; should map via reference |
| `Regulatory ID` | Unmapped | — | — | STRING | — | YES | — | - | 🟡 Required for CBUAE regulatory reporting |
| `Country code` | Mapped | `COUNTRYID` | VARCHAR | CHAR(3) ISO | YES | YES | — | - | ⚠️ Raw source code; should be ISO 3166-1 alpha-3 via reference mapping |
| `Is Secured` | Mapped | `SECURITYTAKEN` (LEA_AGREEMENT_DTL) | VARCHAR | BOOLEAN | YES | YES | — | - | ⚠️ CHAR → BOOLEAN conversion needed |
| `Insurance status` | Mapped | `LAC_COLLATERAL_INS_FLAG_C` | CHAR | STRING (code) | YES | YES | — | - | ⚠️ Raw source code; use reference mapping |
| `Secured amount` | Mapped | `AMOUNT` (LEA_COLLATERAL_GEN_DTL) | NUMBER | DECIMAL(18,4) | YES | YES | — | SCD-2 | ❌ Typed STRING in mapping; must be DECIMAL(18,4); missing `currency_code` |
| `Liquidity score` | Mapped | `LAA_APP_CRSCORE_N` | NUMBER | — | YES | YES | — | - | ❌ Semantic error: credit score ≠ liquidity score; derived metric; violates SLV-007 |
| `Source system code` | Not Available | — | — | STRING | — | NO | Meta | - | ❌ Mandatory metadata column unmapped |
| `Source system ID` | Not mapped | — | — | STRING | — | YES | Meta | - | ❌ Mandatory metadata column absent |
| `IsLogicallyDeleted` | Not mapped | — | — | BOOLEAN | — | YES | Meta | - | ❌ Non-standard name; should be `is_active_flag` (STRING) or `delete_date` (TIMESTAMP) |
| `Create date` | Not mapped | — | — | TIMESTAMP | — | NO | Meta | - | ❌ Not yet mapped; mandatory metadata |
| `Update date` | Not mapped | — | — | TIMESTAMP | — | YES | Meta | - | ❌ Not yet mapped; mandatory metadata |
| `Deleted date` | Not mapped | — | — | TIMESTAMP | — | YES | Meta | - | ❌ Non-standard name; should be `delete_date` |
| `Business Start Date` | — | — | — | TIMESTAMP | — | NO | SCD | SCD-2 | ❌ Non-standard name; should be `effective_from` (TIMESTAMP) |
| `Business End Date` | — | — | — | TIMESTAMP | — | YES | SCD | SCD-2 | ❌ Non-standard name; should be `effective_to` (TIMESTAMP) |
| `Is Active Flag` | — | — | — | STRING | — | NO | SCD | SCD-2 | ❌ Non-standard; should be `is_current` (BOOLEAN) |

**Missing attributes (present in guideline §2.7 but absent from logical model):**
- `run_id` (STRING) — pipeline run identifier
- `source_ingestion_date` (TIMESTAMP) — when source emitted the record
- `collateral_sk` (STRING NOT NULL) — MD5 surrogate key
- `collateral_bk` (STRING NOT NULL) — canonical business key

#### Metadata Columns (SLV-008 / Guideline §2.7)

| Column | Present | Correct Type | Notes |
|---|---|---|---|
| `source_system_code` | ❌ | ❌ | Listed as "Not Available" in mapping |
| `source_system_id` | ❌ | ❌ | Listed without mapping |
| `create_date` | ⚠️ Partial | ❌ | Present in LM but unmapped; type should be TIMESTAMP |
| `update_date` | ⚠️ Partial | ❌ | Present in LM but unmapped; type should be TIMESTAMP |
| `delete_date` | ⚠️ Partial | ❌ | Named `Deleted date` in LM; non-standard; unmapped |
| `is_active_flag` | ⚠️ Partial | ❌ | Named `IsLogicallyDeleted` (inverted logic); BOOLEAN vs STRING |
| `effective_from` | ⚠️ Partial | ❌ | Named `Business Start Date`; CHAR(18) not TIMESTAMP |
| `effective_to` | ⚠️ Partial | ❌ | Named `Business End Date`; CHAR(18) not TIMESTAMP |
| `is_current` | ⚠️ Partial | ❌ | Named `Is Active Flag`; STRING not BOOLEAN |
| `run_id` | ❌ | ❌ | Entirely absent |
| `source_ingestion_date` | ❌ | ❌ | Entirely absent |

#### Findings

##### Finding CollateralAsset-001 — Missing Surrogate Key and Business Key

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | `SLV-001 / SLV-002` — "Every Silver entity must have `<entity>_sk` (MD5 surrogate) + `<entity>_bk` (canonical business key)" (guideline §2.4: "Derive surrogate keys deterministically: `MD5(UPPER(TRIM(business_key)))`") |
| Evidence | Logical model: `bank_logical_model.xlsx` → `Consolidated Metada for Subject`, rows for `Customer_CollateralAsset`; Physical: `collateral_physical_structures.csv` row `customer_collateralasset` |
| Affected Table | `silver.collateral.customer_collateralasset` |
| Affected Column(s) | `collateral_sk` (missing), `collateral_bk` (missing) |
| Confidence | 0.97 |

**Description:** The entity uses `Collateral id` (raw source system NUMBER(8) from `RLS.FIN_LEAM.LEA_COLLATERAL_DTL.COLLATERALID`) as its implicit identifier. No platform surrogate key or canonical business key is defined. Without `collateral_sk`, downstream Gold-layer `fact_collateral_coverage` tables cannot join to the collateral dimension with SCD-2 point-in-time accuracy. FK relationships from sub-type entities and pledge tables cannot be enforced.

**Remediation:** Add `collateral_sk STRING NOT NULL` (MD5 of `UPPER(TRIM(collateral_bk))`) and `collateral_bk STRING NOT NULL` (= source `COLLATERALID` cast to string and canonicalized) to the logical model and physical DDL. Update all ETL to compute the SK before writing.

**Migration Steps:**
1. Add `collateral_bk STRING NOT NULL` and `collateral_sk STRING NOT NULL` as nullable columns initially
2. Backfill: `UPDATE SET collateral_bk = CAST(COLLATERALID AS STRING), collateral_sk = MD5(UPPER(TRIM(CAST(COLLATERALID AS STRING))))`
3. Validate uniqueness of `collateral_sk` and `collateral_bk` (zero duplicates expected)
4. Add NOT NULL constraint to both columns
5. Create CONSTRAINT pk_collateral_asset PRIMARY KEY (collateral_sk)

**Estimated Effort:** Medium  
**Owner:** Data Modeling / ETL

---

##### Finding CollateralAsset-002 — All Logical Data Types CHAR(18)

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | `SLV-009 / SLV-010` — "All timestamps stored in UTC"; "Monetary amounts in smallest unit (fils for AED)"; Guideline §01 Gate 2: "All monetary columns must use DECIMAL(18,4)" |
| Evidence | Logical model: all 29 attributes of `Customer_CollateralAsset` typed as `CHAR(18)` |
| Affected Table | `silver.collateral.customer_collateralasset` |
| Affected Column(s) | All monetary fields: `original_value`, `current_value`, `secured_amount`; date fields: `valuation_date`; boolean fields: `is_secured`, `is_pledged`, `is_active` |
| Confidence | 0.97 |

**Description:** The logical model uniformly assigns `CHAR(18)` to every attribute including monetary amounts, dates, booleans, and codes. This is a modelling placeholder that was never updated. Monetary amounts (`original_value`, `current_value`, `secured_amount`) must be `DECIMAL(18,4)` per DQ standard Gate 2. Date attributes must be `DATE`. Timestamp attributes must be `TIMESTAMP`. Flag/boolean attributes must be `BOOLEAN`.

**Remediation:** Revise the logical model to assign correct canonical types. Update physical DDL to match. Cast source values at ETL time.

**Estimated Effort:** Medium  
**Owner:** Data Modeling

---

##### Finding CollateralAsset-003 — Liquidity Score is a Derived / Incorrectly Sourced Metric

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | `SLV-007` — "No pre-computed metrics in Silver" (Guideline §2.6: "Store only atomic, un-aggregated facts in Silver") |
| Evidence | Mapping sheet `Customer_Collateral_D_Model`, row `Liquidity score` → source `LOS_APP_APPLICATIONS.LAA_APP_CRSCORE_N` |
| Affected Table | `silver.collateral.customer_collateralasset` |
| Affected Column(s) | `liquidity_score` |
| Confidence | 0.90 |

**Description:** `Liquidity score` is mapped from `LAA_APP_CRSCORE_N`, which is the application credit score — not a collateral liquidity score. This is both semantically incorrect and a violation of the no-derived-metrics rule. Collateral liquidity scoring is a derived metric that should be computed in the Gold layer based on asset type, haircut, and market data.

**Remediation:** Remove `liquidity_score` from the Silver entity. If a liquidity classification is needed, store the raw `collateral_type_code` and `haircut_pct` and compute liquidity tiers in Gold.

**Estimated Effort:** Small  
**Owner:** Data Modeling / Business Analysis

---

##### Finding CollateralAsset-004 — Monetary Amounts Missing Currency Code

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | High |
| Guideline Rule | `SLV-010` — "Monetary amounts in smallest unit (fils for AED); `amount_orig` + `currency_code` carried" |
| Evidence | Mapping `Customer_Collateral_D_Model`: `Original value`, `Current value`, `Secured amount` have no corresponding `currency_code` attribute |
| Affected Table | `silver.collateral.customer_collateralasset` |
| Affected Column(s) | `original_value`, `current_value`, `secured_amount` |
| Confidence | 0.95 |

**Description:** Collateral values are multi-currency (real estate in AED, foreign securities in USD/EUR, precious metals in USD). Without `currency_code`, downstream conversion and LTV calculations are impossible. The column naming should also follow the pattern `<measure>_amount_<currency>` for AED amounts or `<measure>_amount_orig` + `currency_code` for multi-currency.

**Remediation:** Add `currency_code CHAR(3) NOT NULL` (ISO 4217). Rename amounts to `original_value_amount_orig`, `current_value_amount_orig`, `secured_amount_orig` with associated `currency_code`. Add `current_value_amount_aed` for AED-equivalent (sourced from FX conversion in Gold, not Silver).

**Estimated Effort:** Medium  
**Owner:** Data Modeling / ETL

---

### 4.2 `Customer_Collateral_Real_Estate`

**State:** 🔴 Incorrect  
**Criticality:** High — Real estate is the most material collateral class for UAE mortgage and SME lending. LTV calculation and CBUAE real estate collateral concentration limits depend on this entity.  
**Confidence:** 0.85  
**Physical Table(s):** `silver.collateral.customer_collateral_real_estate`

#### Industry Fitness

The entity captures UAE real estate pledged as collateral. The attributes (Property ID, Address, Location, Square feet, Lien holder) represent a minimal set. Missing industry-standard attributes include: property registration number (DLD number in UAE), property type (residential/commercial/land), emirate, valuation method (RICS/bank appraiser), and DLD lien registration date. The address is stored as a single concatenated STRING field (`PROPERTY_ADDR1/PROPERTY_ADDR2/PROPERTY_ADDR3`) — this is a normalization violation (unstructured address, see guideline §2.1 Address Normalization).

#### Attribute Review

| Logical Attribute | Mapping Status | Physical Column | Actual Type | Expected Type | Nullable (Actual) | Nullable (Expected) | Key Role | SCD Required | Issues |
|---|---|---|---|---|---|---|---|---|---|
| `Collateral id` | Mapped | `COLLATERALID` (LEA_COLLATERAL_DTL) | NUMBER(8) | STRING (FK → collateral_bk) | NO | NO | FK | - | ❌ Raw source ID; no collateral_sk FK defined |
| `Property id` | Mapped | `PROPERTYID` (LEA_PROPERTY_DTL) | VARCHAR | STRING (BK) | NO | NO | BK | SCD-2 | ❌ No property_sk surrogate |
| `Address` | Mapped | `PROPERTY_ADDR1/2/3` | VARCHAR | Normalized child entity | YES | YES | — | - | ❌ Three source columns concatenated into one STRING; violates 3NF/address normalization guideline |
| `Location` | Mapped | `PROPERTY_ADDR4` | VARCHAR | STRING | YES | YES | — | - | ⚠️ Should map to structured emirate/city/district fields |
| `Square feet` | Mapped | `LAND_AREA` | NUMBER | DECIMAL(10,2) | YES | YES | — | SCD-2 | ⚠️ CHAR(18) in LM; should be DECIMAL(10,2) |
| `Lien holder` | Mapped | `FIRST_LIEN_BY` | VARCHAR | STRING (FK → party_bk) | YES | YES | — | - | ❌ Denormalized party attribute; PII=Y; should be FK to Party entity |
| `Business Start Date` | — | — | — | TIMESTAMP (`effective_from`) | — | NO | SCD | SCD-2 | ❌ Non-standard column name |
| `Business End Date` | — | — | — | TIMESTAMP (`effective_to`) | — | YES | SCD | SCD-2 | ❌ Non-standard column name |
| `Is Active Flag` | — | — | — | BOOLEAN (`is_current`) | — | NO | SCD | SCD-2 | ❌ Non-standard column name and type |

#### Metadata Columns (SLV-008)

| Column | Present | Correct Type | Notes |
|---|---|---|---|
| `source_system_code` | ❌ | ❌ | Absent from this sub-entity |
| `source_system_id` | ❌ | ❌ | Absent |
| `create_date` | ❌ | ❌ | Absent |
| `update_date` | ❌ | ❌ | Absent |
| `delete_date` | ❌ | ❌ | Absent |
| `is_active_flag` | ⚠️ | ❌ | Named `Is Active Flag`; CHAR(18) not STRING/BOOLEAN |
| `effective_from` | ⚠️ | ❌ | Named `Business Start Date`; wrong type |
| `effective_to` | ⚠️ | ❌ | Named `Business End Date`; wrong type |
| `is_current` | ⚠️ | ❌ | Named `Is Active Flag`; wrong type |
| `run_id` | ❌ | ❌ | Absent |
| `source_ingestion_date` | ❌ | ❌ | Absent |

#### Findings

##### Finding RealEstate-001 — Address Concatenation Violates 3NF and Address Normalization Standard

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | `SLV-006` — "3NF within subject area; no derived values"; Guideline §2.1: "Normalize into a child table (one row per item) to preserve all items and support typed semantic fields and SCD semantics" |
| Evidence | Mapping `Customer_Collateral_D_Model` row `Address` → `PROPERTY_ADDR1/PROPERTY_ADDR2/PROPERTY_ADDR3` (three source columns concatenated) |
| Affected Table | `silver.collateral.customer_collateral_real_estate` |
| Affected Column(s) | `address` |
| Confidence | 0.93 |

**Description:** Three source address columns are concatenated into a single unstructured STRING. This prevents: (a) filtering by emirate/city/district, (b) standardization, (c) CBUAE real estate concentration reporting by location. The address should be structured with distinct fields: `address_line_1`, `address_line_2`, `district`, `city`, `emirate_code`, `postal_code`, `country_code`.

**Remediation:** Decompose `Address` into structured fields. Consider a separate `collateral_real_estate_address` child table if multiple addresses per property are possible.

**Estimated Effort:** Medium  
**Owner:** Data Modeling / ETL

---

##### Finding RealEstate-002 — Lien Holder is a Denormalized PII Party Attribute

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | Medium |
| Guideline Rule | `SLV-006` — "3NF within subject area"; Guideline §2.2: "Keep each Silver subject area self-contained" |
| Evidence | Mapping `Customer_Collateral_D_Model` row `Lien holder` → `FIRST_LIEN_BY`; PII = Y |
| Affected Table | `silver.collateral.customer_collateral_real_estate` |
| Affected Column(s) | `lien_holder` |
| Confidence | 0.88 |

**Description:** `lien_holder` (FIRST_LIEN_BY) is a party name — PII data — embedded directly in the collateral entity. This violates 3NF and PII isolation. The lien holder should be referenced via `party_bk` FK to `silver.party`.

**Remediation:** Replace `lien_holder` STRING with `lien_holder_party_bk STRING` (FK to silver.party). Move name resolution to Gold layer.

**Estimated Effort:** Small  
**Owner:** Data Modeling / ETL

---

### 4.3 `Customer_Collateral_Vehicles`

**State:** 🔴 Incorrect  
**Criticality:** Medium — Vehicle collateral is material for personal auto loans and SME fleet financing.  
**Confidence:** 0.83  
**Physical Table(s):** `silver.collateral.customer_collateral_vehicles`

#### Industry Fitness

The entity captures motor vehicle details pledged as collateral. The attribute set (Vehicle ID, Make model, Year, VIN) is minimal but reasonable for a sub-type entity. Missing attributes include: chassis number / registration plate (UAE-specific), registration emirate, estimated mileage, insurance status, and registration expiry date. The concatenation of make and model into a single `Make model` field prevents querying by manufacturer or model separately.

#### Attribute Review

| Logical Attribute | Mapping Status | Physical Column | Actual Type | Expected Type | Nullable | Nullable (Exp) | Key Role | SCD Required | Issues |
|---|---|---|---|---|---|---|---|---|---|
| `Collateral id` | Mapped | `LAC_COLLATERAL_ID_C` (LOS_APP_COLLATERAL) | VARCHAR | STRING (FK → collateral_bk) | NO | NO | FK | - | ❌ Raw source ID; no collateral_sk FK |
| `Vehicle id` | Mapped | `MODELID` (LEA_MODEL_M) | VARCHAR | STRING (BK) | NO | NO | BK | SCD-2 | ❌ Model ID ≠ Vehicle ID; semantic mismatch |
| `Make model` | Mapped | `MODELNO` (LEA_MODEL_M) | VARCHAR | STRING | YES | YES | — | - | ⚠️ Should be split into `vehicle_make` + `vehicle_model` |
| `Year` | Mapped | `YEAR_MODEL` (LEA_MODEL_M) | VARCHAR/INT | INTEGER | YES | YES | — | - | ⚠️ CHAR(18) in LM; should be SMALLINT or INTEGER |
| `VIN` | Unmapped | — | — | STRING | — | YES | — | - | 🟡 UAE Central Bank collateral form requires VIN/chassis no. |
| `Business Start Date` | — | — | — | TIMESTAMP | — | NO | SCD | SCD-2 | ❌ Non-standard name |
| `Business End Date` | — | — | — | TIMESTAMP | — | YES | SCD | SCD-2 | ❌ Non-standard name |
| `Is Active Flag` | — | — | — | BOOLEAN | — | NO | SCD | SCD-2 | ❌ Non-standard name/type |

#### Metadata Columns (SLV-008)

All mandatory metadata columns (`source_system_code`, `source_system_id`, `create_date`, `update_date`, `delete_date`, `run_id`, `source_ingestion_date`) are absent from this sub-entity.

#### Findings

##### Finding Vehicles-001 — Vehicle ID Semantic Mismatch (Model ID ≠ Vehicle Instance)

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | Medium |
| Guideline Rule | Industry best practice: grain must be one row per individual collateral asset instance |
| Evidence | Mapping `Customer_Collateral_D_Model` row `Vehicle id` → `LEA_MODEL_M.MODELID` (a vehicle model reference, not a specific vehicle asset) |
| Affected Table | `silver.collateral.customer_collateral_vehicles` |
| Affected Column(s) | `vehicle_id` |
| Confidence | 0.85 |

**Description:** The source mapping uses `LEA_MODEL_M.MODELID` (a vehicle model/catalogue record) as the Vehicle ID. This captures the vehicle type, not the specific pledged vehicle instance. Two customers pledging the same model car would share the same `vehicle_id`. The correct identifier should be the vehicle's asset ID or registration number from the asset management module.

**Remediation:** Identify the correct source column for the specific vehicle asset ID (likely `LEA_ASSET_M.ASSETID` by analogy with other sub-entities). Revise the mapping.

**Estimated Effort:** Small  
**Owner:** ETL / Business Analysis

---

### 4.4 `Customer_Collateral_Fixed_Deposit`

**State:** 🔴 Incorrect  
**Criticality:** Medium — FD collateral (cash pledge) is highly liquid and commonly used for overdraft and personal loan collateral in UAE banking.  
**Confidence:** 0.83  
**Physical Table(s):** `silver.collateral.customer_collateral_fixed_deposit`

#### Industry Fitness

The entity captures fixed deposit accounts pledged as collateral (a "cash deposit" collateral type under Basel III / CRR). The attribute set (Account number, Bank code, Maturity date, Haircut) is minimal. Missing attributes include: FD account balance, interest rate (required for pre-maturity payoff calculation), lien date, pledge start date, and whether the FD is at the same bank (zero-haircut under Basel III) or a third-party bank. The bank code mapping to a canonical bank identifier (BIC/SWIFT) is not documented.

#### Attribute Review

| Logical Attribute | Mapping Status | Physical Column | Actual Type | Expected Type | Nullable | Nullable (Exp) | Key Role | SCD Required | Issues |
|---|---|---|---|---|---|---|---|---|---|
| `Collateral id` | Mapped | `COLLATERALID` (LEA_GUARANTOR_HIRER_DTL) | NUMBER | STRING (FK) | NO | NO | FK | - | ❌ Wrong source table (guarantor table used for FD FK?); mapping requires verification |
| `Account number` | Mapped | `ACCOUNT_NO` (LOS_DISBURSAL_DTL) | VARCHAR | STRING (BK) | NO | NO | BK | SCD-2 | ⚠️ Disbursement table used — should be from deposit/account master |
| `Bank code` | Mapped | `BANKCD` (LOS_NFIS_FILE_SENT) | VARCHAR | STRING | YES | NO | — | - | ⚠️ NFIS file sent table — unclear if this is the FD-holding bank code |
| `Maturity date` | Mapped | `LAA_MATURITY_DATE` (LOS_APP_APPLICATIONS) | DATE | DATE | YES | NO | — | SCD-2 | ⚠️ Application-level maturity date; may not be FD-specific |
| `Haircut` | Unmapped | — | — | DECIMAL(5,4) | — | YES | — | - | ❌ Unmapped; critical for LTV calculation |
| `Business Start Date` | — | — | — | TIMESTAMP | — | NO | SCD | SCD-2 | ❌ Non-standard name |
| `Business End Date` | — | — | — | TIMESTAMP | — | YES | SCD | SCD-2 | ❌ Non-standard name |
| `Is Active Flag` | — | — | — | BOOLEAN | — | NO | SCD | SCD-2 | ❌ Non-standard name/type |

#### Metadata Columns (SLV-008)

All mandatory metadata columns absent from this sub-entity.

#### Findings

##### Finding FixedDeposit-001 — Source Table Mismatch for Collateral ID and Account Number

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | Medium |
| Guideline Rule | Behavioral rule: "Do not assume an attribute is correctly mapped just because a physical column with a similar name exists — verify type, nullability, and semantics" |
| Evidence | Mapping `Customer_Collateral_D_Model` row `Collateral id` → `LEA_GUARANTOR_HIRER_DTL.COLLATERALID`; row `Account number` → `LOS_DISBURSAL_DTL.ACCOUNT_NO` |
| Affected Table | `silver.collateral.customer_collateral_fixed_deposit` |
| Affected Column(s) | `collateral_id`, `account_number` |
| Confidence | 0.80 |

**Description:** The `Collateral ID` is sourced from the guarantor/hirer detail table rather than from the collateral detail table (as used by all other sub-entities). The `Account number` comes from a disbursement table. Both source tables are questionable for a fixed-deposit collateral entity. The correct source should be the deposit/account master table (e.g., `LEA_ASSET_M` or equivalent FD master).

**Remediation:** Validate source table selection with business analysts. Likely the correct source for FD collateral is the deposit account master, not the disbursement or guarantor tables.

**Estimated Effort:** Small  
**Owner:** Business Analysis / ETL

---

### 4.5 `Customer_Collateral_Precious_Metal`

**State:** 🔴 Incorrect  
**Criticality:** Medium — Gold/precious metal collateral is highly relevant in Gulf banking (CBUAE gold haircut schedules, commodity-linked financing).  
**Confidence:** 0.80  
**Physical Table(s):** `silver.collateral.customer_collateral_precious_metal`

#### Industry Fitness

The entity covers gold, silver, and other precious metals pledged as collateral. In UAE banking, gold collateral is common for Islamic finance structures and SME lending. The model has only `Collateral ID` mapped (1 of 5 business attributes). Four of five core attributes — Metal ID, Karat, Weight, Appraiser ID — are all unmapped ("Not available"). This entity is essentially a stub with no usable collateral-specific content.

#### Attribute Review

| Logical Attribute | Mapping Status | Physical Column | Expected Type | Key Role | Issues |
|---|---|---|---|---|---|
| `Collateral id` | Mapped | `LAC_COLLATERAL_ID_C` | STRING (FK) | FK | ❌ No collateral_sk FK |
| `Metal id` | Unmapped | — | STRING (BK) | BK | ❌ No source identified; no physical table content |
| `Karat` | Unmapped | — | SMALLINT | — | ❌ No source identified |
| `Weight` | Unmapped | — | DECIMAL(10,3) | — | ❌ No source; missing unit (grams/troy oz) |
| `Appraiser id` | Unmapped | — | STRING (FK → party_bk) | FK | ❌ No source; should FK to party |
| All metadata | Absent | — | TIMESTAMP/STRING | Meta | ❌ All 8 standard metadata cols absent |

#### Metadata Columns (SLV-008)

All mandatory metadata columns absent.

#### Findings

##### Finding PreciousMetal-001 — Entity is a Near-Empty Stub (4 of 5 Business Attributes Unmapped)

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | Medium |
| Guideline Rule | `SLV-005` — "DQ-gated writes; quarantine table present" — an entity with no core attributes populated should not be promoted to Silver |
| Evidence | Mapping `Customer_Collateral_D_Model`: Metal id, Karat, Weight, Appraiser id all `Status: Not available` |
| Affected Table | `silver.collateral.customer_collateral_precious_metal` |
| Affected Column(s) | `metal_id`, `karat`, `weight`, `appraiser_id` |
| Confidence | 0.92 |

**Description:** The physical table `customer_collateral_precious_metal` exists but its four core business attributes have no source mapping. The entity cannot support LTV calculation or gold collateral valuation. Deployment in production with no content is a governance and DQ gap. Missing `weight_unit_code` and `purity_pct` are further gaps against CBUAE gold collateral requirements.

**Remediation:** Identify source tables for precious metal details (likely in a leasing/assets master table). Defer production deployment until at least Metal ID, Weight, and Karat are mapped.

**Estimated Effort:** Large  
**Owner:** Business Analysis / ETL

---

### 4.6 `Customer_Collateral_Securities`

**State:** 🔴 Incorrect  
**Criticality:** High — Securities collateral (listed equities, bonds, Sukuk) is subject to daily margin calls and LTV monitoring. Incorrect or missing mapping makes margin monitoring impossible.  
**Confidence:** 0.82  
**Physical Table(s):** `silver.collateral.customer_collateral_securities`

#### Industry Fitness

The entity covers listed securities (equities, bonds, Sukuk) pledged against margin lending facilities. Only `Collateral ID` and `Instrument type` are mapped. `Security ID` (the identifier for the specific security — ISIN or SEDOL), `Ticker`, and `Haircut` are all unmapped. Without `Security ID` (ISIN), real-time valuation via market data feeds and daily haircut application cannot be automated. This entity cannot support margin lending monitoring in its current state.

#### Attribute Review

| Logical Attribute | Mapping Status | Physical Column | Expected Type | Key Role | Issues |
|---|---|---|---|---|---|
| `Collateral id` | Mapped | `LAC_COLLATERAL_ID_C` (LOS_APP_COLLATERAL) | STRING (FK) | FK | ❌ No collateral_sk FK |
| `Security id` | Unmapped | — | STRING (ISIN/SEDOL) | BK | ❌ Critical; ISIN required for market data linkage |
| `Instrument type` | Mapped | `INSTRUMENT` (LEA_INSTRUMENT_DTL) | STRING (code) | — | ⚠️ Raw source code; must map via reference |
| `Ticker` | Unmapped | — | STRING | — | ❌ Unmapped |
| `Haircut` | Unmapped | — | DECIMAL(5,4) | — | ❌ Critical for LTV calculation |
| All metadata | Absent | — | TIMESTAMP/STRING | Meta | ❌ All standard metadata cols absent |

#### Metadata Columns (SLV-008)

All mandatory metadata columns absent.

#### Findings

##### Finding Securities-001 — ISIN/Security ID Unmapped; Entity Cannot Support Margin Monitoring

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | High |
| Guideline Rule | Industry best practice (BIAN Collateral Asset Administration): securities collateral must carry standardized security identifier (ISIN/CUSIP) for automated valuation |
| Evidence | Mapping `Customer_Collateral_D_Model` row `Security ID`: Status = `Not available` |
| Affected Table | `silver.collateral.customer_collateral_securities` |
| Affected Column(s) | `security_id`, `ticker`, `haircut` |
| Confidence | 0.90 |

**Description:** Without `Security ID` (ISIN), this entity cannot be joined to market data feeds for daily mark-to-market valuation. LTV cannot be computed. Haircut schedule application (per CBUAE margin lending framework) is impossible. The entity is functionally incomplete.

**Remediation:** Map `security_id` to the ISIN or internal instrument ID from the securities/instrument master. Add `haircut_pct DECIMAL(5,4)` sourced from the collateral haircut schedule table.

**Estimated Effort:** Large  
**Owner:** Business Analysis / ETL

---

### 4.7 `Customer_Collateral_Guarantees`

**State:** 🔴 Incorrect  
**Criticality:** High — Personal and corporate guarantees are legally enforceable credit enhancements. Under BCBS 272 and UAE credit policy, guarantee details must be accurately tracked for IFRS 9 stage 2/3 assessment.  
**Confidence:** 0.82  
**Physical Table(s):** `silver.collateral.customer_collateral_guarantees`

#### Industry Fitness

The entity captures personal/corporate guarantees as a collateral sub-type. While a guarantee is technically a credit-risk mitigant rather than a physical asset, it is acceptable to model it here as BIAN "Collateral Asset Administration" covers both tangible and intangible collateral including guarantees. However:
1. `Guarantor name` is a denormalized party attribute (PII risk).
2. `Max liability` is a monetary amount (type STRING in mapping).
3. `Expiry date` is unmapped — guarantee expiry is critical for determining protection validity.
4. No guarantee type classification (personal / corporate / bank / government).

#### Attribute Review

| Logical Attribute | Mapping Status | Physical Column | Expected Type | Key Role | Issues |
|---|---|---|---|---|---|
| `Collateral id` | Mapped | `COLLATERALID` (LEA_COLLATERAL_DTL) | STRING (FK) | FK | ❌ No collateral_sk FK |
| `Guarantee id` | Mapped | `GUARANTOR_HIRER_ID` (LEA_PDE_GUARANTOR_HIRER_DTL) | STRING (BK) | BK | ❌ No guarantee_sk surrogate |
| `Guarantor name` | Unmapped | — | STRING | — | ❌ PII party attr; should FK to party entity; not directly stored |
| `Max liability` | Mapped | `COLLATERAL_VALUE` (LEA_COLLATERAL_DTL) | NUMBER | DECIMAL(18,4) | ❌ Sourced from collateral_value (header level); semantic question; typed STRING; missing currency_code |
| `Expiry date` | Unmapped | — | DATE | — | ❌ Unmapped; guarantee validity period critical for credit risk |
| All metadata | Absent | — | TIMESTAMP/STRING | Meta | ❌ All standard metadata cols absent |

#### Metadata Columns (SLV-008)

All mandatory metadata columns absent.

#### Findings

##### Finding Guarantees-001 — Guarantor Name is Denormalized PII; Expiry Date Unmapped

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | High |
| Guideline Rule | `SLV-006` — "3NF within subject area"; DQ Guideline §2 Gate 2: "PII Masking: EID, Card Numbers, Phone Numbers, and Passport Numbers masked via SHA-256" |
| Evidence | Mapping `Customer_Collateral_D_Model` row `Guarantor name`: Status = `Not available`; note PII concern from first-name/surname data |
| Affected Table | `silver.collateral.customer_collateral_guarantees` |
| Affected Column(s) | `guarantor_name`, `expiry_date` |
| Confidence | 0.88 |

**Description:** `Guarantor name` (a full name = PII) must not be stored directly in the collateral entity. The guarantor should already be modelled as a Party in `silver.party`; this entity should carry `guarantor_party_sk` (FK to party). `Expiry date` is entirely unmapped — expired guarantees cannot be identified for IFRS 9 stage assessment.

**Remediation:** Replace `guarantor_name` with `guarantor_party_bk STRING` (FK to silver.party). Map `expiry_date` from the guarantee agreement master table.

**Estimated Effort:** Small  
**Owner:** Data Modeling / ETL / Legal

---

### 4.8 `Customer_Loan_Collateral_Pledges`

**State:** 🔴 Incorrect  
**Criticality:** High — This is the central association table linking collateral assets to loan facilities. It is the foundation of LTV computation and IFRS 9 collateral-coverage calculation.  
**Confidence:** 0.88  
**Physical Table(s):** `silver.collateral.customer_loan_collateral_pledges`

#### Industry Fitness

`Customer_Loan_Collateral_Pledges` models the many-to-many relationship between loans and collateral assets — a critical association entity. The design is directionally correct: a pledge record links a loan account to a collateral asset with a coverage ratio and effective/release dates. However:
1. `Pledge ratio` is ambiguous — the source mapping (`LEA_REPAYMENT_SOURCE_DTL.PLEDGE_AMOUNT`) returns an amount, not a ratio. The name-vs-type mismatch must be resolved.
2. `Release date` is mapped from `MODIFIED_ON` (modification timestamp) — semantically incorrect. Release date should be the contractual collateral release date, not a record modification date.
3. No `loan_sk` FK is present — only a raw `Loan Account number` string. This prevents SCD-2-aware joins between the pledge and loan entities.
4. No `collateral_sk` FK — only `Collateral ID` (source ID). Cannot enforce referential integrity.

#### Attribute Review

| Logical Attribute | Mapping Status | Physical Column | Expected Type | Key Role | SCD Required | Issues |
|---|---|---|---|---|---|---|
| `Pledge id` | Mapped | `APP_ID_C` (LOS_APP_COLLATERAL) | STRING (BK) | BK/PK | SCD-2 | ❌ No pledge_sk surrogate; APP_ID = application ID, not pledge-specific ID |
| `Loan account number` | Mapped | `LAA_APP_ACCOUNTNO_C` (LOS_APP_APPLICATIONS) | STRING (BK) | FK | - | ❌ Should be `loan_bk` → silver.lending.loan; no loan_sk FK |
| `Collateral id` | Mapped | `LAC_COLLATERAL_ID_C` (LOS_APP_COLLATERAL) | STRING (FK) | FK | - | ❌ No collateral_sk FK |
| `Pledge ratio` | Mapped | `PLEDGE_AMOUNT` (LEA_REPAYMENT_SOURCE_DTL) | NUMBER | DECIMAL(5,4) (pct) or DECIMAL(18,4) (amount) | SCD-2 | ❌ Name says "ratio" but source is "amount"; semantic mismatch; CHAR(18) in LM |
| `Effective date` | Mapped | `CREATED_ON` (LOS_APP_COLLATERAL) | DATE | DATE | - | ⚠️ CHAR(18) in LM; creation date ≈ effective date but should be verified |
| `Release date` | Mapped | `MODIFIED_ON` (LOS_APP_COLLATERAL) | DATE | DATE | - | ❌ Modification timestamp ≠ collateral release date; semantic error |
| `Net secured value` | Mapped | `lac_collateral_value_n` (LOS_APP_COLLATERAL) | NUMBER | DECIMAL(18,4) | SCD-2 | ❌ Pre-computed value — potentially violates SLV-007; missing currency_code |
| `Business Start Date` | — | — | TIMESTAMP (`effective_from`) | — | SCD-2 | ❌ Non-standard |
| `Business End Date` | — | — | TIMESTAMP (`effective_to`) | — | SCD-2 | ❌ Non-standard |
| `Is Active Flag` | — | — | BOOLEAN (`is_current`) | — | SCD-2 | ❌ Non-standard |

#### Metadata Columns (SLV-008)

All mandatory metadata columns absent from this entity.

#### Findings

##### Finding Pledges-001 — Pledge Ratio vs. Amount Semantic Mismatch; Release Date Wrong Source

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | Behavioral rule: "Do not assume correct mapping based on similar column names — verify type, nullability, and semantics" |
| Evidence | Mapping `Customer_Collateral_D_Model` row `Pledge ratio` → `PLEDGE_AMOUNT` (an amount, not a ratio); `Release date` → `MODIFIED_ON` (modification timestamp) |
| Affected Table | `silver.collateral.customer_loan_collateral_pledges` |
| Affected Column(s) | `pledge_ratio`, `release_date` |
| Confidence | 0.88 |

**Description:** `Pledge ratio` is mapped from `PLEDGE_AMOUNT`, a monetary amount field. The entity intends to store a coverage ratio (e.g., 0.75 = 75% coverage), but the source is an amount. Either the column name is wrong or the mapping is wrong. `Release date` is sourced from `MODIFIED_ON` (a record modification timestamp), which is not the contractual collateral release date. These two errors will produce incorrect LTV and collateral-coverage calculations.

**Remediation:** Clarify with business analysts whether `pledge_ratio` should be an amount or a percentage. If it's a percentage, find the correct source column. Separately identify the contractual release date column for `release_date`.

**Estimated Effort:** Small  
**Owner:** Business Analysis / ETL

---

### 4.9 `Customer_Collateral_Receivables`

**State:** 🔴 Incorrect  
**Criticality:** High — Trade receivables collateral (invoice discounting, factoring) is material for SME and corporate lending. Debtor concentration and invoice validity are key credit risk parameters.  
**Confidence:** 0.85  
**Physical Table(s):** `silver.collateral.customer_collateral_receivables`

#### Industry Fitness

The entity models receivables (invoices/trade debts) pledged as collateral — a common SME lending structure. The design covers Collateral ID, Receivable ID, Debtor details, Invoice amount, Due date, and Concentration limit. `Debtor name` is denormalized (should FK to Party). The `Concentration limit` field sourced from `LOS_APP_ELIGIBILITY.MAX_LIMIT_N` is a loan-application-level limit, not a per-receivable concentration metric — semantic mismatch.

#### Attribute Review

| Logical Attribute | Mapping Status | Physical Column | Expected Type | Issues |
|---|---|---|---|---|
| `Collateral id` | Mapped | `COLLATERALID` (LEA_COLLATERAL_DTL) | STRING (FK) | ❌ No collateral_sk FK |
| `Receivable id` | Mapped | `INVOICENUM` (LEA_ASSET_M) | STRING (BK) | ❌ No receivable_sk surrogate |
| `Debtor id` | Unmapped | — | STRING (FK → party_bk) | ❌ No source found; key FK field |
| `Debtor name` | Unmapped | — | Should NOT be stored | ❌ PII party attr; should FK to party; name should not be in Silver |
| `Invoice amount` | Mapped | `AMTFIN` (LEA_ASSET_M) | DECIMAL(18,4) | ❌ Typed STRING in LM; missing currency_code |
| `Due date` | Mapped | `DUEDATE` (LEA_REPAYSCH_DTL) | DATE | ⚠️ CHAR(18) in LM; must be DATE |
| `Concentration limit` | Mapped | `MAX_LIMIT_N` (LOS_APP_ELIGIBILITY) | DECIMAL(18,4) | ❌ Application-level limit ≠ debtor concentration; semantic error; typed STRING |
| All metadata | Absent | — | TIMESTAMP/STRING | ❌ All standard metadata cols absent |

#### Metadata Columns (SLV-008)

All mandatory metadata columns absent.

---

### 4.10 `Customer_Collateral_Crypto`

**State:** 🔴 Incorrect  
**Criticality:** Low — Crypto is not currently a mainstream CBUAE-recognized collateral class. Included for completeness.  
**Confidence:** 0.75  
**Physical Table(s):** `silver.collateral.customer_collateral_crypto`

#### Industry Fitness

Cryptocurrency as collateral is a nascent class. The CBUAE does not currently recognize crypto as eligible collateral under standard lending frameworks. The inclusion of this entity is forward-looking. `Volatility index` and `Custodian ID` are both unmapped — these are the two most important attributes (custodian for regulated custody of crypto assets; volatility for dynamic haircut scheduling). The `Coin type` sourced from `CURRENCYID` (a currency master field) suggests crypto coins are mapped using the currency reference table — ambiguous but potentially workable for major coins (BTC/ETH).

#### Attribute Review

| Logical Attribute | Mapping Status | Expected Type | Issues |
|---|---|---|---|
| `Collateral id` | Mapped | STRING (FK) | ❌ No collateral_sk FK |
| `Wallet id` | Mapped (`ASSETID`) | STRING (BK) | ⚠️ Asset ID used as wallet ID — verify semantic correctness |
| `Coin type` | Mapped (`CURRENCYID`) | STRING (code) | ⚠️ Currency master used for crypto — may not include all token types |
| `Volatility index` | Unmapped | DECIMAL(10,4) | ❌ No source; required for dynamic haircut |
| `Custodian id` | Unmapped | STRING (FK → party_bk) | ❌ No source; regulatory requirement for licensed custodian |
| All metadata | Absent | TIMESTAMP/STRING | ❌ All standard metadata cols absent |

#### Metadata Columns (SLV-008)

All mandatory metadata columns absent.

---

### 4.11 `Customer_Collateral_IP`

**State:** 🔴 Incorrect  
**Criticality:** Low — Intellectual property collateral is rare in UAE retail banking.  
**Confidence:** 0.78  
**Physical Table(s):** **None — no physical table exists**

#### Industry Fitness

Intellectual property (patents, trademarks, copyrights) as collateral is relatively uncommon but growing in SME innovation finance. The entity has reasonable attributes (IP ID, ID type, Registration number, Royalty income). No physical table has been deployed. This entity exists only in the logical model.

#### Attribute Review

| Logical Attribute | Mapping Status | Expected Type | Issues |
|---|---|---|---|
| `Collateral id` | Mapped (LM) | STRING (FK) | ❌ No physical table; no collateral_sk |
| `Ip id` | Mapped (`ASSETID` LEA_PROPERTY_DTL) | STRING (BK) | ⚠️ Property detail table used for IP — verify semantic correctness |
| `Id type` | Mapped (`LAC_COLLATERAL_TYPE_C`) | STRING (code) | ⚠️ Generic collateral type code used; IP-specific type sub-codes needed |
| `Registration number` | Mapped (`LAT_TRADEIN_REGISTRATION_NO_C`) | STRING | ⚠️ Trade-in registration used for IP — questionable mapping |
| `Royalty income 12months` | Unmapped | DECIMAL(18,4) | ❌ Key income-based valuation metric; unmapped |
| All metadata | Absent | TIMESTAMP/STRING | ❌ All metadata cols absent |

#### Metadata Columns (SLV-008)

All metadata columns absent (no physical table).

#### Findings

##### Finding IP-001 — No Physical Table Deployed

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | Low |
| Guideline Rule | Guideline §2.11: "Create and approve the logical and physical model in Erwin before generating DDL. DDL must be generated from the approved model." |
| Evidence | Physical structures CSV does not contain any table matching `customer_collateral_ip` or equivalent |
| Affected Table | `silver.collateral.customer_collateral_ip` (missing) |
| Affected Column(s) | All |
| Confidence | 0.99 |

**Description:** No physical Delta table exists for this logical entity. All attributes are effectively unmapped at the physical layer.

**Remediation:** Create physical table DDL from the logical model once the entity is fully mapped and reviewed. Defer until source mappings are confirmed.

**Estimated Effort:** Small  
**Owner:** Data Modeling / DBA

---

### 4.12 `Customer_Collateral_Inventory`

**State:** 🔴 Incorrect  
**Criticality:** Low — Inventory collateral is relevant for trade/commodity finance.  
**Confidence:** 0.78  
**Physical Table(s):** **None — no physical table exists**

#### Industry Fitness

Inventory (stock, goods in transit, warehouse receipts) as collateral is relevant for SME working capital loans and trade finance. The entity models Inventory ID, type, turnover ratio, and obsolescence risk. Four of five business attributes are unmapped. No physical table exists.

#### Attribute Review

| Logical Attribute | Mapping Status | Expected Type | Issues |
|---|---|---|---|
| `Collateral id` | Mapped (LM) | STRING (FK) | ❌ No physical table; no collateral_sk |
| `Inventory id` | Unmapped | STRING (BK) | ❌ No source found |
| `Inventory type` | Unmapped | STRING (code) | ❌ No source found |
| `Turnover ratio` | Unmapped | DECIMAL(10,4) | ❌ Derived metric; violates SLV-007 if pre-computed |
| `Obsolescence risk` | Unmapped | STRING (code) | ❌ No source found |
| All metadata | Absent | TIMESTAMP/STRING | ❌ All metadata cols absent |

#### Metadata Columns (SLV-008)

All metadata columns absent (no physical table).

#### Findings

##### Finding Inventory-001 — No Physical Table and Near-Complete Unmapped State

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | Low |
| Guideline Rule | `SLV-005` — entity should not be deployed without DQ-gated source mappings |
| Evidence | Physical structures CSV: no `customer_collateral_inventory` table; Mapping: 4/5 business attrs `Not available` |
| Affected Table | `silver.collateral.customer_collateral_inventory` (missing) |
| Affected Column(s) | All |
| Confidence | 0.99 |

**Description:** Entity exists only in the logical model. No physical table and no source mappings for core attributes. `Turnover ratio` if implemented would violate SLV-007 (derived metric). Defer deployment until source identification is complete.

**Remediation:** Identify source system for inventory collateral (likely a trade finance or warehouse management module). Map all attributes before creating the physical table.

**Estimated Effort:** Large  
**Owner:** Business Analysis / Data Modeling

---

## 5. Denormalization Register

| ID | Tables Involved | Logical Origins | Classification | Correctness Risk | Impact | Recommendation | Confidence |
|---|---|---|---|---|---|---|---|
| DEN-001 | `customer_collateralasset` | `Customer_CollateralAsset.customer_number_cif` ← Party entity | Unnecessary | High | Customer CIF stored in collateral header; any customer data change requires update in collateral table | Replace with `party_bk STRING` FK to `silver.party`; resolve name at Gold | 0.90 |
| DEN-002 | `customer_collateral_real_estate` | `Real_Estate.lien_holder` ← Party entity | Unnecessary | High | Lien holder (party name, PII) embedded in collateral sub-type; violates PII isolation | Replace with `lien_holder_party_bk STRING` FK to `silver.party` | 0.88 |
| DEN-003 | `customer_collateral_guarantees` | `Guarantees.guarantor_name` ← Party entity | Unnecessary | High | Guarantor name (PII) stored directly; not FK to party entity | Replace with `guarantor_party_bk STRING` FK to `silver.party` | 0.90 |
| DEN-004 | `customer_collateral_receivables` | `Receivables.debtor_name` ← Party entity | Unnecessary | Medium | Debtor name embedded; PII; no party FK | Replace with `debtor_party_bk STRING` FK to `silver.party` | 0.85 |
| DEN-005 | `customer_loan_collateral_pledges` | `Pledges.net_secured_value` — computed from `current_value × (1 - haircut)` | Unnecessary | Medium | Pre-computed metric in Silver violates SLV-007; stale if inputs change | Remove from Silver; compute in Gold `fact_collateral_coverage` | 0.82 |

**DEN-001 Justification:** None documented. Customer CIF in collateral header adds no query benefit that cannot be served by a Gold-layer join via `collateral_sk → party_sk`. Classify as **Unnecessary**; recommend re-normalization.

**DEN-002 Justification:** None documented. Lien holder resolution can be done at Gold layer. **Unnecessary.**

**DEN-003 Justification:** None documented. Guarantor is a Party with their own Party record. **Unnecessary.**

**DEN-004 Justification:** None documented. Debtor should be a Party (corporate counterparty). **Unnecessary.**

**DEN-005 Justification:** None documented. Net secured value = current_value × (1 - effective_haircut). This is a derived metric that changes with every valuation or haircut update; it must not be persisted in Silver. **Unnecessary.**

---

## 6. Guideline Compliance Summary

| Rule Code | Rule Title | Status | Violation Count | Worst Severity |
|---|---|---|---|---|
| SLV-001 | `<entity>_sk` MD5 surrogate key on every entity | ❌ Fail | 12 | P0 |
| SLV-002 | `<entity>_bk` canonical business key on every entity | ❌ Fail | 12 | P0 |
| SLV-003 | SCD-2 default; standard `effective_from`/`effective_to`/`is_current` | ❌ Fail | 12 | P1 |
| SLV-004 | No source-system codes; use `silver.reference.code_mapping` | ❌ Fail | 6 | P1 |
| SLV-005 | DQ-gated writes; quarantine table | ⚠️ Partial | 2 entities no physical table | P1 |
| SLV-006 | 3NF; no derived values or aggregations | ❌ Fail | 5 denormalization instances | P1 |
| SLV-007 | No pre-computed metrics in Silver | ❌ Fail | 2 (`liquidity_score`, `net_secured_value`) | P0 |
| SLV-008 | All metadata columns present and correctly typed | ❌ Fail | 12 | P1 |
| SLV-009 | All timestamps in UTC | ⚠️ Partial | Non-standard column names prevent verification | P1 |
| SLV-010 | Monetary amounts in smallest unit; `amount_orig` + `currency_code` | ❌ Fail | 8 monetary columns without `currency_code` | P0 |
| NAM-001 | snake_case, lowercase, no reserved words | ❌ Fail | 12 (PascalCase attrs in LM: IsPledged, IsActive, etc.) | P2 |
| NAM-003 | Silver table naming: `<entity>` (no `customer_` prefix) | ❌ Fail | 12 tables with redundant `customer_` prefix | P2 |
| Guideline §2.1 | 3NF address normalization | ❌ Fail | 1 (address concatenation in Real Estate) | P0 |
| Guideline §2.11 | Erwin model before DDL; no non-model tables | ❌ Fail | 7 non-production tables in schema | P0 |
| Guideline §2.12 | Informatica GDGC catalog registration | ⚠️ Unknown | Cannot verify from available artifacts | P2 |

---

## 7. Remediation Plan

### Prioritized Action List

| # | Action | Entity / Table | Priority | Effort | Owner | Dependency |
|---|---|---|---|---|---|---|
| 1 | Remove 2 mis-scoped transaction tables from collateral schema | `non_electronic_channel_transaction__*` | P0 | Small | DBA | None |
| 2 | Drop / archive 7 non-production tables (backup/test/dummy/wbg) | All `*_backup`, `*_test`, `*_dummy`, `*_wbg` | P0 | Small | DBA | None |
| 3 | Add `collateral_sk` (MD5) + `collateral_bk` to `customer_collateralasset` | `customer_collateralasset` | P0 | Medium | Data Modeling / ETL | None |
| 4 | Add `<subtype>_sk` + `collateral_bk` FK to all sub-type entities | All 11 sub-type/pledge tables | P0 | Medium | Data Modeling / ETL | Action 3 |
| 5 | Correct all CHAR(18) types in logical model and physical DDL | All 12 entities | P0 | Medium | Data Modeling | None |
| 6 | Add `currency_code` to all monetary amount columns | `customer_collateralasset`, `customer_loan_collateral_pledges`, `customer_collateral_guarantees`, `customer_collateral_receivables` | P0 | Small | Data Modeling / ETL | Action 5 |
| 7 | Decompose concatenated `Address` field into structured fields | `customer_collateral_real_estate` | P0 | Medium | ETL | None |
| 8 | Remove `liquidity_score` and `net_secured_value` from Silver entities | `customer_collateralasset`, `customer_loan_collateral_pledges` | P0 | Small | Data Modeling / ETL | None |
| 9 | Standardize all SCD metadata column names (`effective_from`, `effective_to`, `is_current`) | All 12 entities | P1 | Medium | Data Modeling / ETL | None |
| 10 | Add all missing standard metadata columns (`source_system_code`, `run_id`, `source_ingestion_date`, etc.) to all 10 sub-type entities | All sub-type entities except `customer_collateralasset` | P1 | Medium | Data Modeling / ETL | Action 9 |
| 11 | Replace `guarantor_name`, `lien_holder`, `debtor_name` party attrs with `*_party_bk` FKs | `customer_collateral_guarantees`, `customer_collateral_real_estate`, `customer_collateral_receivables` | P1 | Small | Data Modeling / ETL | None |
| 12 | Fix `release_date` mapping in pledges (MODIFIED_ON → correct contractual release column) | `customer_loan_collateral_pledges` | P1 | Small | ETL / Business Analysis | BA confirmation |
| 13 | Fix `pledge_ratio` vs. `pledge_amount` semantic ambiguity | `customer_loan_collateral_pledges` | P1 | Small | ETL / Business Analysis | BA confirmation |
| 14 | Implement `silver.reference.code_mapping` lookups for `collateral_type_code`, `lien_priority_code`, `instrument_type_code`, `country_code` | `customer_collateralasset`, `customer_collateral_securities`, `customer_collateral_vehicles` | P1 | Medium | ETL | Reference Data SA readiness |
| 15 | Map unmapped attributes for Precious Metal, Securities, Crypto, Inventory, IP | All 5 stub entities | P1 | Large | Business Analysis / ETL | Source system discovery |
| 16 | Create `collateral_valuation` entity for IFRS 9 LTV history tracking | New entity required | P1 | Large | Data Modeling / ETL | Action 3 |
| 17 | Create `collateral_lien_registration` entity (DLD registration, lien status tracking) | New entity required | P1 | Medium | Data Modeling / ETL | Party SA linkage |
| 18 | Create physical tables for `Customer_Collateral_IP` and `Customer_Collateral_Inventory` | Two missing tables | P1 | Small | DBA / Data Modeling | Source mappings (Action 15) |
| 19 | Rename tables to drop `customer_` prefix per NAM-003 | All 12 tables | P2 | Medium | DBA | Consumer pipeline updates |
| 20 | Rename PascalCase columns to snake_case per NAM-001 | All entities | P2 | Medium | Data Modeling / DBA | Consumer pipeline updates |
| 21 | Register all entities in Informatica GDGC catalog | All 12 entities | P2 | Small | Data Governance | None |
| 22 | Fix Vehicle ID semantic mapping (MODELID → actual vehicle asset ID) | `customer_collateral_vehicles` | P2 | Small | ETL / Business Analysis | Source confirmation |

### Suggested Schedule

| Sprint | Actions |
|---|---|
| Sprint 1 (immediate / P0) | Actions 1, 2 (schema cleanup); Action 3–4 (SK/BK); Action 5–6 (data types + currency); Action 7 (address decomposition); Action 8 (remove derived metrics) |
| Sprint 2 (P1 — metadata & FK) | Actions 9, 10 (SCD metadata standardization); Action 11 (party FK normalization); Actions 12, 13 (pledge mapping fixes) |
| Sprint 3 (P1 — mappings & entities) | Action 14 (reference code mapping); Action 15 (unmapped stub entities); Actions 16, 17 (new entities: valuation history, lien registration) |
| Sprint 4 (P1/P2 — physical completion) | Action 18 (create missing tables); Actions 19, 20 (rename for naming compliance) |
| Sprint 5 (P2 — governance) | Action 21 (catalog registration); Action 22 (vehicle ID fix) |

---

## Appendix

### A. Logical → Physical Mapping Summary

| Logical Entity | Logical Attribute | Physical Table | Physical Column | Mapping Status |
|---|---|---|---|---|
| Customer_CollateralAsset | Collateral id | customer_collateralasset | COLLATERALID (LEA_COLLATERAL_DTL) | Mapped |
| Customer_CollateralAsset | Customer number (CIF) | customer_collateralasset | CIF_NO (NBFC_COUNTRY_M) | Mapped |
| Customer_CollateralAsset | Collateral type | customer_collateralasset | LAC_COLLATERAL_TYPE_C (LOS_APP_COLLATERAL) | Mapped |
| Customer_CollateralAsset | Original value | customer_collateralasset | LAC_COLLATERAL_VALUE_N (LOS_APP_COLLATERAL) | Mapped |
| Customer_CollateralAsset | Current value | customer_collateralasset | N_MARKET_VALUE (LEA_PROPERTY_DTL) | Mapped |
| Customer_CollateralAsset | Valuation date | customer_collateralasset | APPRAISAL_DATE (LEA_PROPERTY_DTL) | Mapped |
| Customer_CollateralAsset | Lien priority | customer_collateralasset | LAC_LIEN_TITLE_FLAG_C (LOS_APP_COLLATERAL) | Mapped |
| Customer_CollateralAsset | Country code | customer_collateralasset | COUNTRYID (NBFC_COUNTRY_M) | Mapped |
| Customer_CollateralAsset | Is Secured | customer_collateralasset | SECURITYTAKEN (LEA_AGREEMENT_DTL) | Mapped |
| Customer_CollateralAsset | Insurance status | customer_collateralasset | LAC_COLLATERAL_INS_FLAG_C (LOS_APP_COLLATERAL) | Mapped |
| Customer_CollateralAsset | Secured amount | customer_collateralasset | AMOUNT (LEA_COLLATERAL_GEN_DTL) | Mapped |
| Customer_CollateralAsset | Liquidity score | customer_collateralasset | LAA_APP_CRSCORE_N (LOS_APP_APPLICATIONS) | Mapped (Incorrect — semantic error) |
| Customer_CollateralAsset | Owner group id | customer_collateralasset | — | Unmapped |
| Customer_CollateralAsset | Collateral source | customer_collateralasset | — | Unmapped |
| Customer_CollateralAsset | Base haircut | customer_collateralasset | — | Unmapped |
| Customer_CollateralAsset | Risk adjustment | customer_collateralasset | — | Unmapped |
| Customer_CollateralAsset | Effective haircut | customer_collateralasset | — | Unmapped |
| Customer_CollateralAsset | Is Pledged | customer_collateralasset | — | Unmapped |
| Customer_CollateralAsset | Is Active | customer_collateralasset | — | Unmapped |
| Customer_CollateralAsset | Regulatory ID | customer_collateralasset | — | Unmapped |
| Customer_CollateralAsset | Source system code | customer_collateralasset | — | Not Available |
| Customer_CollateralAsset | Source system ID | customer_collateralasset | — | Not mapped |
| Customer_CollateralAsset | IsLogicallyDeleted | customer_collateralasset | — | Not mapped |
| Customer_CollateralAsset | Create date | customer_collateralasset | — | Not mapped |
| Customer_CollateralAsset | Update date | customer_collateralasset | — | Not mapped |
| Customer_CollateralAsset | Deleted date | customer_collateralasset | — | Not mapped |
| Customer_Collateral_Real_Estate | Collateral id | customer_collateral_real_estate | COLLATERALID (LEA_COLLATERAL_DTL) | Mapped |
| Customer_Collateral_Real_Estate | Property id | customer_collateral_real_estate | PROPERTYID (LEA_PROPERTY_DTL) | Mapped |
| Customer_Collateral_Real_Estate | Address | customer_collateral_real_estate | PROPERTY_ADDR1/2/3 (LEA_PROPERTY_DTL) | Mapped (Concatenated — 3NF violation) |
| Customer_Collateral_Real_Estate | Location | customer_collateral_real_estate | PROPERTY_ADDR4 (LEA_PROPERTY_DTL) | Mapped |
| Customer_Collateral_Real_Estate | Square feet | customer_collateral_real_estate | LAND_AREA (LEA_PROPERTY_DTL) | Mapped |
| Customer_Collateral_Real_Estate | Lien holder | customer_collateral_real_estate | FIRST_LIEN_BY (LEA_PROPERTY_DTL) | Mapped (Denormalized PII) |
| Customer_Collateral_Vehicles | Collateral id | customer_collateral_vehicles | LAC_COLLATERAL_ID_C (LOS_APP_COLLATERAL) | Mapped |
| Customer_Collateral_Vehicles | Vehicle id | customer_collateral_vehicles | MODELID (LEA_MODEL_M) | Mapped (Semantic issue: model ≠ vehicle) |
| Customer_Collateral_Vehicles | Make model | customer_collateral_vehicles | MODELNO (LEA_MODEL_M) | Mapped |
| Customer_Collateral_Vehicles | Year | customer_collateral_vehicles | YEAR_MODEL (LEA_MODEL_M) | Mapped |
| Customer_Collateral_Vehicles | VIN | customer_collateral_vehicles | — | Unmapped |
| Customer_Collateral_Fixed_Deposit | Collateral id | customer_collateral_fixed_deposit | COLLATERALID (LEA_GUARANTOR_HIRER_DTL) | Mapped (Source table suspect) |
| Customer_Collateral_Fixed_Deposit | Account number | customer_collateral_fixed_deposit | ACCOUNT_NO (LOS_DISBURSAL_DTL) | Mapped (Source table suspect) |
| Customer_Collateral_Fixed_Deposit | Bank code | customer_collateral_fixed_deposit | BANKCD (LOS_NFIS_FILE_SENT) | Mapped |
| Customer_Collateral_Fixed_Deposit | Maturity date | customer_collateral_fixed_deposit | LAA_MATURITY_DATE (LOS_APP_APPLICATIONS) | Mapped |
| Customer_Collateral_Fixed_Deposit | Haircut | customer_collateral_fixed_deposit | — | Unmapped |
| Customer_Collateral_Precious_Metal | Collateral id | customer_collateral_precious_metal | LAC_COLLATERAL_ID_C (LOS_APP_COLLATERAL) | Mapped |
| Customer_Collateral_Precious_Metal | Metal id | customer_collateral_precious_metal | — | Unmapped |
| Customer_Collateral_Precious_Metal | Karat | customer_collateral_precious_metal | — | Unmapped |
| Customer_Collateral_Precious_Metal | Weight | customer_collateral_precious_metal | — | Unmapped |
| Customer_Collateral_Precious_Metal | Appraiser id | customer_collateral_precious_metal | — | Unmapped |
| Customer_Collateral_Securities | Collateral id | customer_collateral_securities | LAC_COLLATERAL_ID_C (LOS_APP_COLLATERAL) | Mapped |
| Customer_Collateral_Securities | Security id | customer_collateral_securities | — | Unmapped |
| Customer_Collateral_Securities | Instrument type | customer_collateral_securities | INSTRUMENT (LEA_INSTRUMENT_DTL) | Mapped (Raw source code) |
| Customer_Collateral_Securities | Ticker | customer_collateral_securities | — | Unmapped |
| Customer_Collateral_Securities | Haircut | customer_collateral_securities | — | Unmapped |
| Customer_Collateral_Guarantees | Collateral id | customer_collateral_guarantees | COLLATERALID (LEA_COLLATERAL_DTL) | Mapped |
| Customer_Collateral_Guarantees | Guarantee id | customer_collateral_guarantees | GUARANTOR_HIRER_ID (LEA_PDE_GUARANTOR_HIRER_DTL) | Mapped |
| Customer_Collateral_Guarantees | Guarantor name | customer_collateral_guarantees | — | Unmapped |
| Customer_Collateral_Guarantees | Max liability | customer_collateral_guarantees | COLLATERAL_VALUE (LEA_COLLATERAL_DTL) | Mapped (Semantic question) |
| Customer_Collateral_Guarantees | Expiry date | customer_collateral_guarantees | — | Unmapped |
| Customer_Loan_Collateral_Pledges | Pledge id | customer_loan_collateral_pledges | APP_ID_C (LOS_APP_COLLATERAL) | Mapped (APP_ID ≠ Pledge ID) |
| Customer_Loan_Collateral_Pledges | Loan account number | customer_loan_collateral_pledges | LAA_APP_ACCOUNTNO_C (LOS_APP_APPLICATIONS) | Mapped |
| Customer_Loan_Collateral_Pledges | Collateral id | customer_loan_collateral_pledges | LAC_COLLATERAL_ID_C (LOS_APP_COLLATERAL) | Mapped |
| Customer_Loan_Collateral_Pledges | Pledge ratio | customer_loan_collateral_pledges | PLEDGE_AMOUNT (LEA_REPAYMENT_SOURCE_DTL) | Mapped (Semantic error: amount ≠ ratio) |
| Customer_Loan_Collateral_Pledges | Effective date | customer_loan_collateral_pledges | CREATED_ON (LOS_APP_COLLATERAL) | Mapped |
| Customer_Loan_Collateral_Pledges | Release date | customer_loan_collateral_pledges | MODIFIED_ON (LOS_APP_COLLATERAL) | Mapped (Semantic error: modification ≠ release) |
| Customer_Loan_Collateral_Pledges | Net secured value | customer_loan_collateral_pledges | lac_collateral_value_n (LOS_APP_COLLATERAL) | Mapped (Derived metric violation SLV-007) |
| Customer_Collateral_Receivables | Collateral id | customer_collateral_receivables | COLLATERALID (LEA_COLLATERAL_DTL) | Mapped |
| Customer_Collateral_Receivables | Receivable id | customer_collateral_receivables | INVOICENUM (LEA_ASSET_M) | Mapped |
| Customer_Collateral_Receivables | Debtor id | customer_collateral_receivables | — | Unmapped |
| Customer_Collateral_Receivables | Debtor name | customer_collateral_receivables | — | Unmapped |
| Customer_Collateral_Receivables | Invoice amount | customer_collateral_receivables | AMTFIN (LEA_ASSET_M) | Mapped |
| Customer_Collateral_Receivables | Due date | customer_collateral_receivables | DUEDATE (LEA_REPAYSCH_DTL) | Mapped |
| Customer_Collateral_Receivables | Concentration limit | customer_collateral_receivables | MAX_LIMIT_N (LOS_APP_ELIGIBILITY) | Mapped (Semantic error) |
| Customer_Collateral_Crypto | Collateral id | customer_collateral_crypto | COLLATERALID (LEA_COLLATERAL_DTL) | Mapped |
| Customer_Collateral_Crypto | Wallet id | customer_collateral_crypto | ASSETID (LEA_ASSET_M) | Mapped |
| Customer_Collateral_Crypto | Coin type | customer_collateral_crypto | CURRENCYID (LEA_COLLATERAL_GEN_DTL) | Mapped |
| Customer_Collateral_Crypto | Volatility index | customer_collateral_crypto | — | Unmapped |
| Customer_Collateral_Crypto | Custodian id | customer_collateral_crypto | — | Unmapped |
| Customer_Collateral_IP | All | None (no physical table) | — | All Unmapped |
| Customer_Collateral_Inventory | All | None (no physical table) | — | All Unmapped |

---

### B. Guideline Citations

| Rule Code | Source File | Full Rule Text |
|---|---|---|
| SLV-001/002 | `guidelines/11-modeling-dos-and-donts.md` §2.4 | "Derive surrogate keys deterministically: `MD5(UPPER(TRIM(business_key)))` for single-component keys; `MD5(CONCAT_WS('|', key_part_1, key_part_2))` for composite keys. Do NOT use auto-increment sequences, `UUID()`, `NEWID()`, or `RAND()`-based surrogate keys." |
| SLV-003 | `guidelines/11-modeling-dos-and-donts.md` §2.3 | "Track all changes to tracked attributes with SCD Type 2: add a new row with updated values, close the prior row (`effective_to` = change timestamp, `is_current` = FALSE). Every Silver entity table must have `effective_from`, `effective_to`, `is_current`." |
| SLV-004 | `guidelines/11-modeling-dos-and-donts.md` §2.5 | "Resolve all source-system codes to canonical platform values using a central code-mapping reference (e.g., `silver.reference.code_mapping`). Store the canonical code, not the raw source code, in entity columns." |
| SLV-005 | `guidelines/01-data-quality-controls.md` §3.2 | "Failed records routed to quarantine / error tables rather than discarded. All DQ results must be logged and auditable." |
| SLV-006 | `guidelines/11-modeling-dos-and-donts.md` §2.1 | "Decompose entities into 3NF: every non-key attribute depends on the whole primary key and nothing but the primary key. Repeating groups must become child tables." |
| SLV-007 | `guidelines/11-modeling-dos-and-donts.md` §2.6 | "Store only atomic, un-aggregated facts in Silver. Do NOT pre-compute totals, averages, counts, ratios, or any derived metrics in Silver tables." |
| SLV-008 | `guidelines/11-modeling-dos-and-donts.md` §2.7 | "Include the following Silver technical audit columns on every Silver entity table: `source_system_code`, `source_system_id`, `create_date`, `update_date`, `delete_date`, `is_active_flag`, `effective_from`, `effective_to`, `is_current`, `run_id`, `source_ingestion_date`." |
| SLV-009 | `guidelines/11-modeling-dos-and-donts.md` §2.9 | "Store all timestamps in UTC. Convert source-local timestamps at ingestion." |
| SLV-010 | `guidelines/01-data-quality-controls.md` Gate 2 | "All monetary columns must use `DECIMAL(18,4)`." |
| NAM-001 | `guidelines/06-naming-conventions.md` | "Use snake_case for all identifiers (catalogs, schemas, tables, columns, views). Use lowercase only — no mixed case." |
| NAM-003 | `guidelines/06-naming-conventions.md` | "Silver entity table pattern: `<entity>` (e.g., `customer`, `account`, `transaction`)." |
| DQ Gate 2 | `guidelines/01-data-quality-controls.md` §2 | "Standardisation — Currencies: All monetary columns must use `DECIMAL(18,4)`. PII Masking: EID, Card Numbers, Phone Numbers, and Passport Numbers masked via SHA-256 unless stored in a restricted container." |
| Guideline §2.11 | `guidelines/11-modeling-dos-and-donts.md` §2.11 | "Create and approve the logical and physical model in Erwin (or equivalent modeling tool) before generating or writing DDL. DDL must be generated from the approved model. Tables that exist in the platform but have no corresponding entity in the Erwin model are a governance gap." |

---

### C. Industry Standard References

| Reference | Standard | Relevance |
|---|---|---|
| BIAN v11 "Collateral Asset Administration" | BIAN Banking Industry Architecture Network v11 | Defines canonical boundaries for collateral management service domain; validates entity set scope |
| IFRS 9 §5.5 ECL | International Financial Reporting Standards 9 | Requires LTV and collateral-coverage calculation for Stage 2/3 assessment; drives need for `collateral_valuation` history entity and `Customer_Loan_Collateral_Pledges` accuracy |
| BCBS 239 Principle 6 | Basel Committee on Banking Supervision 239 | "Adaptable" — data aggregation must be accurate and complete for credit risk collateral data; drives haircut decomposition requirement |
| CRR Article 194-198 | EU Capital Requirements Regulation (applicable as CBUAE equivalent) | Defines eligible collateral classes and haircut methodologies for regulatory capital; drives `base_haircut`, `risk_adjustment`, `effective_haircut` attributes |
| CBUAE Mortgage Regulations | Central Bank UAE Real Estate Finance Regulations | LTV caps on UAE real estate collateral require structured address data (emirate) and DLD registration number |
| IFW (IBM Industry Framework for Banking) | IBM/Teradata Industry Framework for Banking | `FINANCIAL_COLLATERAL` subject area pattern: supertype/subtype model for collateral asset classes; validates the CollateralAsset header + sub-type design |
| Basel III / CRR Art. 207 | Basel III Collateral Recognition | Cash (Fixed Deposit) collateral from the same bank receives 0% haircut; requires bank code reference to distinguish in-house vs. third-party FD |
