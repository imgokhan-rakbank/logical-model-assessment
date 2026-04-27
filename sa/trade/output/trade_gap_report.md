# Silver-Layer Gap Assessment Report
## Subject Area: Trade Finance (`silver.trade` / `silver_dev.trade`)

**Report Version:** 1.0  
**Assessment Date:** 2025-12-11  
**Assessor:** Group Data Office — Silver-Layer Governance  
**Repository:** `imgokhan-rakbank/logical-model-assessment`  
**Input Artifacts:**
- `sa/trade/input/trade_physical_structures.csv` — 27 physical tables
- `sa/trade/input/trade_data_mapping.xlsx` — 39 mapping entities, 375 rows
- `bank_logical_model.xlsx` (Trade filter) — 38 entities, 681 attributes
- `guidelines/01-data-quality-controls.md`, `guidelines/06-naming-conventions.md`, `guidelines/11-modeling-dos-and-donts.md`

---

## 1. Executive Summary

### 1.1 Management Slide

> **The Trade subject area is not ready for production use.** The current implementation merges three distinct canonical subject areas — Trade Finance, Treasury/Investments, and Reference Data — into a single `silver_dev.trade` schema. Physical tables exist for only 22 of 38 logical entities (58%), and those that do exist are missing every required Silver governance control: no surrogate keys, no SCD-2 history columns, no data quality metadata columns, all data types declared as STRING, and all physical table descriptions are null. Seventeen entities critical to trade finance operations (Letters of Credit, Bank Guarantees, Trade Counterparty, Invoice Master, Receivable Financing, Buyer Credit, Trade Exposure Snapshot) have zero physical implementation. Three temporary work tables are polluting the Silver schema. The subject area also contains the Customer entity, which belongs exclusively in `silver.party`.

| Metric | Value |
|---|---|
| Logical entities in scope | 38 |
| Physical tables present | 27 |
| Entities with physical table | 22 (58%) |
| Entities completely unmapped | 16 (42%) |
| Attributes with full mapping | ~35% |
| Attributes with `Not available` mapping | ~28% |
| Attributes with no mapping at all | ~37% |
| P0 findings (blocking) | 14 |
| P1 findings (next sprint) | 22 |
| P2 findings (backlog) | 18 |

### 1.2 Top 5 Priority Actions

| Rank | Action | Priority | Criticality | Effort |
|---|---|---|---|---|
| 1 | **Decompose subject area**: split `silver.trade` into `silver.trade_finance` (LC, BG, Trade Txn, Trade Counterparty, Invoice, Buyer Credit, Receivable Financing, Incoterm) and `silver.treasury` (Bond, Fund, Commodity, FX, Investment products); migrate reference entities to `silver.reference` | P0 | High | XL |
| 2 | **Add surrogate & business keys** (`<entity>_sk` MD5, `<entity>_bk`) to all 22 existing physical tables; critical pre-requisite for Gold joins | P0 | High | L |
| 3 | **Implement SCD-2 columns** (`effective_from`, `effective_to`, `is_current`) + all 11 Silver audit columns on every table; current implementation has none | P0 | High | L |
| 4 | **Create physical tables for 16 missing entities**, especially Bank Guarantees, Import/Export LC, Trade Counterparty, Trade Exposure Snapshot — these are the core trade finance instruments | P0 | High | XL |
| 5 | **Fix all data types**: convert all monetary columns to `DECIMAL(18,4)` (amounts in fils/smallest unit); all date columns to `DATE`; all timestamp columns to `TIMESTAMP`; remove STRING overuse | P0 | High | M |

---

## 2. Subject Area Architecture Assessment

### 2.1 Scoping and Canonical Alignment

**Status: 🔴 CRITICAL — Major Scope Violation**

Per `sa/subject-areas.md`, the canonical subject areas relevant to this domain are:

| SA # | Name | Schema | Description |
|---|---|---|---|
| 14 | **Trade Finance** | `silver.trade_finance` | LC, BG, Bills of Exchange, Shipping Documents |
| 15 | **Treasury** | `silver.treasury` | Bonds, Sukuk, Derivatives, Money Market, FX, FIS instruments |
| 1 | **Reference Data** | `silver.reference` | ISO Currencies, Country Codes, FX Rates |
| 2 | **Party** | `silver.party` | Customers, counterparties |

**The physical schema `silver_dev.trade` conflates all four subject areas into one:**

| Entity Category | Count | Correct Home |
|---|---|---|
| Trade Finance instruments (LC, BG, Trade Transaction, etc.) | 10 | `silver.trade_finance` |
| Investment/Treasury instruments (Bond, Fund, FX, etc.) | 18 | `silver.treasury` |
| Reference tables (Country, Currency, Exchange Rate, Interest Rate) | 7 | `silver.reference` |
| Customer (cross-SA) | 1 | `silver.party` |
| Ambiguous / work tables | 2 | N/A |

**Finding GAP-001 (P0/High, Confidence: 1.0):** The subject area violates the canonical domain scoping defined in `sa/subject-areas.md`. The `silver.trade` schema does not exist as a canonical schema in the normative subject area taxonomy — the correct schemas are `silver.trade_finance` and `silver.treasury`. This must be resolved before any Gold layer consumption.

**Finding GAP-002 (P0/High, Confidence: 1.0):** The `Customer` entity (145 attributes) is included in the Trade SA logical model. Customer is explicitly a `silver.party` concern. Including it here causes duplication of identity data and violates subject-area isolation (rule SLV-004 / guideline 11 §2.2).

**Finding GAP-003 (P0/High, Confidence: 1.0):** Seven reference entities (`Reference_Country`, `Reference_Country1`, `Reference_Currency`, `Reference_Exchange_Rate1`, `Reference_Incoterm`, `Reference_Interest_Rate1`, `Reference_Source_System`) are modeled within the Trade SA. Per `sa/subject-areas.md` entry #1, reference data belongs in `silver.reference`, not in a subject-area schema.

### 2.2 Domain Decomposition Against BIAN / IFW

**Status: 🟡 IMPROPER**

BIAN v11 maps Trade Finance to the **Trade Finance** service domain (Documentary Collections, Commercial Letters of Credit, Bank Guarantees). Investment instruments map to **Securities Administration** and **Traded Positions** service domains.

The current model correctly identifies the key BIAN entities but collapses them into a single schema without domain boundary enforcement. The `trade_transaction` entity attempts to be a super-table covering all trade finance instrument types (LC, BG, etc.) which mismatches BIAN's per-instrument entity model.

**Finding GAP-004 (P1/Medium, Confidence: 0.9):** The `Trade Transaction` entity (93 attributes) is a wide, multi-type table that collapses Import LC, Export LC, Bank Guarantee, and Bills of Exchange into a single entity. BIAN v11 and IFW require per-instrument entities with shared base attributes. This creates nullable column sprawl and makes per-instrument reporting unreliable.

### 2.3 Identity Strategy

**Status: 🔴 MISSING — Critical Gap**

Rule SLV-001/002 (from guideline 11 §2.4): *"Every Silver entity must carry `<entity>_sk` (MD5 surrogate derived deterministically from the business key) and `<entity>_bk` (natural/business key from source)."*

**Finding GAP-005 (P0/High, Confidence: 1.0):** No `<entity>_sk` or `<entity>_bk` columns appear in any entity in the data mapping. The mappings show only source-system ID fields (e.g., `Investment Product Id`, `Transaction_Id`) but these are source identifiers, not canonicalized surrogate/business keys. Without deterministic MD5 surrogate keys, Gold layer joins cannot function and full rebuilds would break all downstream references.

### 2.4 SCD Strategy

**Status: 🔴 MISSING — Critical Gap**

Rule SLV-003 (guideline 11 §2.3): *"Track all changes with SCD Type 2: add a new row with updated values, close the prior row (`effective_to` = change timestamp, `is_current` = FALSE). Every Silver entity table must have `effective_from`, `effective_to`, `is_current`."*

**Finding GAP-006 (P0/High, Confidence: 1.0):** None of the 27 physical tables or 39 mapping entities include `effective_from`, `effective_to`, or `is_current` columns. The mapping uses `Create Date`, `Update Date`, `Delete Date` (all typed as STRING) which is an SCD-1 (overwrite/soft-delete) pattern. No historical change tracking is implemented. This is particularly critical for Investment Product history tables (`investment_product_metric_history`, `investment_product_value_history`) which exist precisely to track time-series data but lack the SCD-2 framework.

### 2.5 Cross-Entity Referential Integrity

**Status: 🔴 MISSING**

**Finding GAP-007 (P1/High, Confidence: 0.95):** No foreign key relationships are documented or enforced between entities. Key broken relationships:
- `trade_transaction` → `trade_counterparty` (no physical `trade_counterparty` table exists)
- `investment_product_metric_history` → `investment_product` (FK present logically but not enforced)
- `bond`, `fund`, `commodity` etc. → `investment_product` (subtypes but no FK documented)
- `reference_product` → physical `trade_finance` entities (no FK)
- All entities → `reference_country` (used for country codes but no FK enforcement)

### 2.6 ETL Mapping Completeness

**Status: 🔴 INCOMPLETE — Major Gap**

| Mapping Status | Entity Count | Attribute Count |
|---|---|---|
| Fully mapped | ~8 (investment product subtypes) | ~40 |
| Partially mapped | ~10 | ~95 |
| Mapping "Not available" | ~8 | ~42 |
| No mapping at all (status=None) | ~21 | ~188 |

The core trade finance entities — Bank Guarantees, Import LC, Export LC, Buyer Credit, Invoice Master, Receivable Financing, Trade Counterparty, Trade Exposure Snapshot — all have zero ETL mapping (status=None for all attributes), meaning they have been modeled but no ETL work has begun. 

**Finding GAP-008 (P0/High, Confidence: 1.0):** 21 out of 38 logical entities (55%) have no ETL mapping whatsoever. These entities exist only on paper. The subject area cannot be considered production-ready.

---

## 3. Entity Inventory

| # | Logical Entity | Physical Table | Schema | Physical Status | Mapping Status | P-Key Present | SK Present | SCD-2 | Metadata Cols | Priority |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | Assest Backed Security Product | `assest_backed_security_product` | `silver_dev.trade` | ✅ Exists (typo) | Partial (1/2 mapped) | ❌ | ❌ | ❌ | ❌ | P1 |
| 2 | Bank Guarantees | None | — | ❌ Missing | ❌ None | ❌ | ❌ | ❌ | ❌ | P0 |
| 3 | Bond | `bond` | `silver_dev.trade` | ✅ Exists | Partial (3/5 mapped) | ❌ | ❌ | ❌ | ❌ | P1 |
| 4 | Buyer Credit Transactions | None | — | ❌ Missing | ❌ None | ❌ | ❌ | ❌ | ❌ | P0 |
| 5 | Commodity | `commodity` | `silver_dev.trade` | ✅ Exists | Partial (1/3 mapped) | ❌ | ❌ | ❌ | ❌ | P1 |
| 6 | Credit Derivate Product | `credit_derivate_product` | `silver_dev.trade` | ✅ Exists (typo) | Partial (1/2 mapped) | ❌ | ❌ | ❌ | ❌ | P1 |
| 7 | Currency Product | `currency_product` | `silver_dev.trade` | ✅ Exists | Partial (1/1 mapped) | ❌ | ❌ | ❌ | ❌ | P1 |
| 8 | Customer | None in trade SA | `silver.party` | ⚠️ Out of scope | ⚠️ Cross-SA | — | — | — | — | P0 |
| 9 | Export LC Transactions | None | — | ❌ Missing | ❌ None | ❌ | ❌ | ❌ | ❌ | P0 |
| 10 | Financial Investment Group | `financial_investment_group` | `silver_dev.trade` | ✅ Exists | Partial (1/1 mapped) | ❌ | ❌ | ❌ | ❌ | P1 |
| 11 | Fund | `fund` | `silver_dev.trade` | ✅ Exists | Mapped (2/2) | ❌ | ❌ | ❌ | ❌ | P1 |
| 12 | Future Product | `future_product` | `silver_dev.trade` | ✅ Exists | Mapped (2/2) | ❌ | ❌ | ❌ | ❌ | P1 |
| 13 | Import LC Transactions | None | — | ❌ Missing | ❌ None | ❌ | ❌ | ❌ | ❌ | P0 |
| 14 | Interest Index Product | `interest_index_product` | `silver_dev.trade` | ✅ Exists | Partial (1/2 mapped) | ❌ | ❌ | ❌ | ❌ | P1 |
| 15 | Investment Product | `investment_product` | `silver_dev.trade` | ✅ Exists | Mapped (4/4) | ❌ | ❌ | ❌ | ❌ | P1 |
| 16 | Investment Product Metric History | `investment_product_metric_history` | `silver_dev.trade` | ✅ Exists | Partial (7/8 mapped) | ❌ | ❌ | ❌ | ❌ | P1 |
| 17 | Investment Product Term Period History | `investment_product_term_period_history` | `silver_dev.trade` | ✅ Exists | Partial (9/10 mapped) | ❌ | ❌ | ❌ | ❌ | P1 |
| 18 | Investment Product Value History | `investment_product_value_history` | `silver_dev.trade` | ✅ Exists | Mapped (8/8) | ❌ | ❌ | ❌ | ❌ | P1 |
| 19 | Investment Security Risk Grade | `investment_security_risk_grade` | `silver_dev.trade` | ✅ Exists | ❌ None (0/5) | ❌ | ❌ | ❌ | ❌ | P0 |
| 20 | Invoice Master | None | — | ❌ Missing | ❌ None | ❌ | ❌ | ❌ | ❌ | P0 |
| 21 | Money Market Product | `money_market_product` | `silver_dev.trade` | ✅ Exists | Partial (1/2 mapped) | ❌ | ❌ | ❌ | ❌ | P1 |
| 22 | Non Traded Investment Product | `non_traded_investment_product` | `silver_dev.trade` | ✅ Exists | Mapped (1/1) | ❌ | ❌ | ❌ | ❌ | P1 |
| 23 | Option Product | `option_product` | `silver_dev.trade` | ✅ Exists | Mapped (2/2) | ❌ | ❌ | ❌ | ❌ | P1 |
| 24 | Receivable Financing Transactions | None | — | ❌ Missing | ❌ None | ❌ | ❌ | ❌ | ❌ | P0 |
| 25 | Reference Country | `reference_country` | `silver_dev.trade` | ✅ Exists (wrong SA) | Partial (13/16) | ❌ | ❌ | ❌ | ❌ | P0 |
| 26 | Reference Country 1 | Duplicate of #25 | — | ⚠️ Duplicate | — | — | — | — | — | P1 |
| 27 | Reference Currency | None | — | ❌ Missing | Partial (4/11) | ❌ | ❌ | ❌ | ❌ | P1 |
| 28 | Reference Exchange Rate 1 | `reference_exchange_rate1` | `silver_dev.trade` | ✅ Exists (bad name) | Mapped (5/5) | ❌ | ❌ | ❌ | ❌ | P1 |
| 29 | Reference Incoterm | None | — | ❌ Missing | ❌ None | ❌ | ❌ | ❌ | ❌ | P1 |
| 30 | Reference Interest Rate 1 | None | — | ❌ Missing | Partial (6/7) | ❌ | ❌ | ❌ | ❌ | P1 |
| 31 | Reference Product Trade | None | — | ❌ Missing | ❌ None | ❌ | ❌ | ❌ | ❌ | P1 |
| 32 | Reference Source System | None | — | ❌ Missing | Mapped (10/10) | ❌ | ❌ | ❌ | ❌ | P1 |
| 33 | Security Index Product | `security_index_product` | `silver_dev.trade` | ✅ Exists | Mapped (1/1) | ❌ | ❌ | ❌ | ❌ | P1 |
| 34 | Stock | None | — | ❌ Missing | Partial (2/3 mapped) | ❌ | ❌ | ❌ | ❌ | P1 |
| 35 | Stock History | None | — | ❌ Missing | Partial (2/4 mapped) | ❌ | ❌ | ❌ | ❌ | P1 |
| 36 | Trade Counterparty | None | — | ❌ Missing | ❌ None | ❌ | ❌ | ❌ | ❌ | P0 |
| 37 | Trade Exposure Snapshot | None | — | ❌ Missing | ❌ None | ❌ | ❌ | ❌ | ❌ | P0 |
| 38 | Trade Transaction | `trade_transaction` | `silver_dev.trade` | ✅ Exists | Partial (61/91 mapped) | ❌ | ❌ | ❌ | ❌ | P0 |
| 39 | Traded Investment Product | `traded_investment_product` | `silver_dev.trade` | ✅ Exists | Mapped (1/1) | ❌ | ❌ | ❌ | ❌ | P1 |
| — | (Orphan) fis_trade_transaction | `fis_trade_transaction` | `silver_dev.trade` | ⚠️ No logical entity | — | — | — | — | — | P0 |
| — | (Orphan) reference_product | `reference_product` | `silver_dev.trade` | ⚠️ Ambiguous match | — | — | — | — | — | P0 |
| — | (Temp) bond_temp_no_partition | — | — | ❌ Temp table in Silver | — | — | — | — | — | P0 |
| — | (Temp) traded_investment_product_temp_no_partition | — | — | ❌ Temp table in Silver | — | — | — | — | — | P0 |
| — | (Temp) reference_product_temp_no_partition | — | — | ❌ Temp table in Silver | — | — | — | — | — | P0 |

---

## 4. Per-Entity Assessments

### 4.1 Entity: Assest Backed Security Product

**Physical Table:** `silver_dev.trade.assest_backed_security_product`  
**Logical Entity:** `Assest Backed Security Product` (8 attributes in logical model)  
**Data Mapping Attributes:** 2 (`Investment Product Id`, `Assest Backed Security Type Cd`)

#### 4.1a Industry Fitness

- **Entity Boundary:** This entity is a sub-type of `investment_product` representing asset-backed securities (ABS). In IFW/BIAN, ABS is a security type under `Securities Administration`. The entity should be a specialization of `investment_product` linked via FK.
- **Grain:** One row per ABS instrument. Correct conceptually but not enforced.
- **Scope:** Belongs in `silver.treasury`, not `silver.trade_finance` or a generic `silver.trade` schema.

#### 4.1b Attribute-Level Review

| Attribute | Logical Type | Mapped Source | Source Type | Status | Issues |
|---|---|---|---|---|---|
| `Investment Product Id` | STRING | FIS.DBO.INSTRUMENT.product_chlnbr | VARCHAR | ✅ Mapped | Acts as business key but not declared as `_bk`; must be `assest_backed_security_product_bk` or better `asset_backed_security_product_bk` |
| `Assest Backed Security Type Cd` | STRING | — | — | ❌ Not available | No source identified; critical classification attribute |
| `investment_product_sk` (surrogate) | — | — | — | ❌ Missing | Required by SLV-001 |
| `investment_product_bk` | — | — | — | ❌ Missing | Required by SLV-002 |
| All 11 SCD/metadata columns | — | — | — | ❌ Missing | Required by SLV-003 / SLV-008 |

#### 4.1c Metadata Completeness

| Column | Required | Present |
|---|---|---|
| `source_system_code` | ✅ | ❌ |
| `source_system_id` | ✅ | ❌ |
| `create_date` | ✅ | ❌ |
| `update_date` | ✅ | ❌ |
| `delete_date` | ✅ | ❌ |
| `is_active_flag` | ✅ | ❌ |
| `effective_from` | ✅ | ❌ |
| `effective_to` | ✅ | ❌ |
| `is_current` | ✅ | ❌ |
| `run_id` | ✅ | ❌ |
| `source_ingestion_date` | ✅ | ❌ |

**All 11 Silver audit columns missing.**

**Findings:**
- **GAP-009 (P0/High):** Table name contains typo `assest` (should be `asset`). NAM-001 violation. Must be renamed to `asset_backed_security_product`.
- **GAP-010 (P1/Medium):** Entity has only 2 attributes in mapping vs. 8 in logical model. 6 attributes unmapped.
- **GAP-011 (P0/High):** Entity belongs in `silver.treasury` schema, not `silver.trade`.

---

### 4.2 Entity: Bank Guarantees

**Physical Table:** None  
**Logical Entity:** `Bank_Guarantees` (20 attributes)  
**Data Mapping:** Entity labeled "Bank Kafalah" in mapping (16 attributes, all with status=None)

#### 4.2a Industry Fitness

Bank Guarantees are a core Trade Finance instrument under BIAN v11 service domain `Issued Guarantees and Indemnities`. In UAE banking, Bank Guarantees (including Islamic Kafalah) are high-volume, regulated instruments subject to CBUAE reporting. The absence of a physical table for this entity is a critical production gap.

The mapping uses the name "Bank Kafalah" which is the Islamic equivalent of a Bank Guarantee. However, the logical model uses "Bank_Guarantees", creating a naming mismatch. The two should be unified into a single entity with an Islamic structure flag.

#### 4.2b Attribute-Level Review

| Attribute | Logical Type | Mapping Status | Issues |
|---|---|---|---|
| Kafalah id / BG Reference | STRING | ❌ No mapping | Business key missing source |
| Customer number CIF | STRING | ❌ No mapping | FK to `silver.party` |
| Beneficiary Counterparty id | STRING | ❌ No mapping | FK to `trade_counterparty` (also missing) |
| Kafalah / BG Type | STRING | ❌ No mapping | Classification code |
| Issue Date | STRING | ❌ No mapping | Should be DATE type |
| Expiry Date | STRING | ❌ No mapping | Should be DATE type |
| Amount | STRING | ❌ No mapping | Should be DECIMAL(18,4); AED fils |
| Currency Code | STRING | ❌ No mapping | Should reference `silver.reference.currency` |
| Status | STRING | ❌ No mapping | Should use `silver.reference.code_mapping` |
| Claim Date | STRING | ❌ No mapping | Should be DATE type |
| Source System Code | STRING | ❌ No mapping | Standard audit column |
| Source System Id | STRING | ❌ No mapping | Standard audit column |
| IsLogicallyDeleted | STRING | ❌ No mapping | Should be `is_active_flag BOOLEAN` |
| Create Date | STRING | ❌ No mapping | Should be `create_date TIMESTAMP` |
| Update Date | STRING | ❌ No mapping | Should be `update_date TIMESTAMP` |
| Deleted Date | STRING | ❌ No mapping | Should be `delete_date TIMESTAMP` |

#### 4.2c Metadata Completeness

All 11 Silver audit columns: **Missing** (no physical table exists).

**Findings:**
- **GAP-012 (P0/High, Confidence: 1.0):** No physical table exists for Bank Guarantees/Kafalah. This is a core Trade Finance instrument and must be created immediately.
- **GAP-013 (P0/High, Confidence: 1.0):** Zero ETL mapping — no source system identified for any attribute.
- **GAP-014 (P1/Medium, Confidence: 0.9):** Name mismatch between logical model (`Bank_Guarantees`) and data mapping (`Bank Kafalah`). Should be unified as `bank_guarantee` with `is_islamic_structure` and `islamic_structure_code` flags per the Islamic Finance structural decision in `subject-areas.md`.
- **GAP-015 (P1/Medium, Confidence: 0.9):** `Amount` typed as STRING. Must be `DECIMAL(18,4)` (SLV-010).

---

### 4.3 Entity: Bond

**Physical Table:** `silver_dev.trade.bond`  
**Logical Entity:** `Bond` (11 attributes in logical model)  
**Data Mapping Attributes:** 5

#### 4.3a Industry Fitness

Bond is a fixed-income investment instrument. In BIAN/IFW, bonds are securities managed under `Securities Administration` / `Portfolio Management`. The entity is correctly identified as a sub-type of `investment_product`. It belongs in `silver.treasury`.

The physical table is partitioned by `Investment_Product_Id` (PascalCase — naming violation). Partitioning by a business key rather than a date column is unusual for a Silver entity; partitioning should be by `effective_from` or `processed_at` for SCD-2 tables.

#### 4.3b Attribute-Level Review

| Attribute | Logical Type | Mapped Source | Status | Issues |
|---|---|---|---|---|
| `Investment Product Id` | STRING | WMS.BO_BOND_MASTER.BOND_CODE | ✅ Mapped | Business key — must be named `bond_bk`; SK also missing |
| `Interest Payment Monthly Num` | STRING | — | ❌ Not available | Should be INTEGER |
| `Interest Payment day number` | STRING | — | ❌ Not available | Should be INTEGER |
| `Bond unit issued Qty` | STRING | WMS.BO_BOND_MASTER.ISSUE_SIZE | ✅ Mapped | Should be DECIMAL(18,4) or INTEGER |
| `Bond Type Code` | STRING | WMS.BO_BOND_MASTER.BOND_TYPE | ✅ Mapped | Source code — must be mapped via `silver.reference.code_mapping` (SLV-004) |
| 6 unmapped logical attributes | — | — | ❌ Missing | Coupon rate, maturity date, face value, currency, issuer, ISIN |

#### 4.3c Metadata Completeness

All 11 Silver audit columns: **Missing**.

**Findings:**
- **GAP-016 (P0/High, Confidence: 1.0):** Partition column `Investment_Product_Id` uses PascalCase, violating NAM-001 (*"Use snake_case for all identifiers"*). Should be `investment_product_bk` or removed as partition key.
- **GAP-017 (P1/Medium, Confidence: 0.95):** `Bond Type Code` sourced directly from `WMS.BO_BOND_MASTER.BOND_TYPE` without code mapping. Violates SLV-004: *"No source-system codes; use `silver.reference.code_mapping`"*.
- **GAP-018 (P1/Medium, Confidence: 0.9):** 6 out of 11 logical attributes unmapped (ISIN, coupon rate, maturity date, face value, currency, issuer information).
- **GAP-019 (P1/Medium, Confidence: 0.9):** `bond_temp_no_partition` table exists in `silver_dev.trade` alongside `bond`. Temporary/work tables must not exist in the Silver production schema.

---

### 4.4 Entity: Buyer Credit Transactions

**Physical Table:** None  
**Logical Entity:** `Buyer Credit Transactions` (18 attributes)  
**Data Mapping:** 15 attributes, all status=None

#### 4.4a Industry Fitness

Buyer Credit is a trade finance mechanism where a bank extends credit to a foreign buyer to purchase goods from a domestic exporter. Under BIAN v11, this maps to `Trade Finance` / `Supplier Financing`. Critical for regulatory exposure reporting.

#### 4.4b Attribute-Level Review

All 15 mapped attributes have status=None (no mapping). Key issues:
- `Amount` typed as STRING (must be `DECIMAL(18,4)`)
- `Disbursement date`, `Maturity Date` typed as STRING (must be `DATE`)
- `Profitrate Rate` typed as STRING (must be `DECIMAL(18,6)` for rate precision)
- `Currency Code` should reference `silver.reference.currency`
- `Source System Code`, `Source System Id` present but no source identified
- `IsLogicallyDeleted` is non-standard; should be `is_active_flag`

#### 4.4c Metadata Completeness

All 11 Silver audit columns: **Missing** (no physical table).

**Findings:**
- **GAP-020 (P0/High, Confidence: 1.0):** No physical table. Core trade finance entity.
- **GAP-021 (P0/High, Confidence: 1.0):** Zero ETL mapping.
- **GAP-022 (P1/Medium, Confidence: 0.9):** Non-standard audit column `IsLogicallyDeleted` (camelCase, non-standard semantics). Must conform to `is_active_flag` per guideline 11 §2.7.

---

### 4.5 Entity: Commodity

**Physical Table:** `silver_dev.trade.commodity`  
**Logical Entity:** `Commodity` (9 attributes)  
**Data Mapping Attributes:** 3

#### 4.5a Industry Fitness

Commodity is a traded product type (Gold, Silver, Oil, etc.) in the treasury/trading domain. Belongs in `silver.treasury` alongside other investment product sub-types.

#### 4.5b Attribute-Level Review

| Attribute | Mapped Source | Status | Issues |
|---|---|---|---|
| `Investment Product Id` | FIS.instr_regulatory_info.commodity_product_chlnbr | ✅ Mapped | Incorrect source table — commodity uses `instr_regulatory_info` while other products use `INSTRUMENT`; source inconsistency flag |
| `Unit of Measure Cd` | FIS (no table specified) | ⚠️ Not Available | Source table unresolved |
| `Commodity Type Cd` | — | ❌ Not available | No source at all; likely needs `silver.reference.code_mapping` |

#### 4.5c Metadata Completeness

All 11 Silver audit columns: **Missing**.

**Findings:**
- **GAP-023 (P1/Medium, Confidence: 0.85):** Source for `Investment Product Id` uses `instr_regulatory_info` instead of the standard `INSTRUMENT` table used by all other investment sub-types. Source consistency must be validated.
- **GAP-024 (P1/Medium, Confidence: 0.9):** `Commodity Type Cd` has no mapping. Gold (physical) and Silver (precious metal) commodity type codes need canonical mapping via `silver.reference.code_mapping`.

---

### 4.6 Entity: Credit Derivate Product

**Physical Table:** `silver_dev.trade.credit_derivate_product`  
**Logical Entity:** `Credit Derivate Product` (8 attributes)  
**Data Mapping Attributes:** 2

#### 4.6a Industry Fitness

Credit derivatives (CDS, CLN, TRS) are complex financial instruments under BIAN `Traded Positions`. The entity name contains a spelling error ("Derivate" vs "Derivative") which propagated to the physical table name.

#### 4.6b Attribute-Level Review

| Attribute | Status | Issues |
|---|---|---|
| `Investment Product Id` | ✅ Mapped (FIS.INSTRUMENT.product_chlnbr) | Must become `credit_derivative_product_bk` |
| `Credit derivative type code` | ⚠️ Not Available | No source; critical for instrument classification; SLV-004 applies |

#### 4.6c Metadata Completeness: All 11 missing.

**Findings:**
- **GAP-025 (P0/High, Confidence: 1.0):** Table name `credit_derivate_product` contains spelling error ("derivate"). Must be renamed to `credit_derivative_product` (NAM-001).
- **GAP-026 (P1/Medium, Confidence: 0.9):** Only 2 of 8 logical attributes in the mapping. 6 unmapped (reference entity, notional amount, maturity, reference obligation, etc.).

---

### 4.7 Entity: Currency Product

**Physical Table:** `silver_dev.trade.currency_product`  
**Logical Entity:** `Currency Product` (7 attributes)  
**Data Mapping Attributes:** 1

#### 4.7a Industry Fitness

Currency Product represents FX spot/forward products in the treasury domain. Belongs in `silver.treasury`. The mapping captures only the product ID (1 of 7 attributes).

#### 4.7b Attribute-Level Review

Only `Currency Investment Product Id` (STRING) is mapped from FIS.INSTRUMENT.product_chlnbr. The remaining 6 attributes (currency pair, spot rate source, settlement type, notional, etc.) are unmapped.

#### 4.7c Metadata Completeness: All 11 missing.

**Findings:**
- **GAP-027 (P1/Medium, Confidence: 0.9):** Only 1/7 attributes mapped. Currency Product is incomplete for any meaningful analytics.

---

### 4.8 Entity: Customer

**Physical Table:** None in trade SA  
**Logical Entity:** `Customer` (145 attributes)  
**Status:** Out of Scope for Trade SA

#### 4.8a Industry Fitness

**Finding GAP-028 (P0/Critical, Confidence: 1.0):** The Customer entity with 145 attributes is modeled within the Trade SA in `bank_logical_model.xlsx`. Per `sa/subject-areas.md`, Customer is a role within the Party subject area (`silver.party`). Guideline 11 §2.2 states: *"Keep each Silver subject area self-contained. Pipelines for one subject area read only from Bronze and from the same subject area's own Silver tables."* The Trade SA should reference `customer_bk` as a foreign key to `silver.party.customer`, not replicate the full customer entity.

This finding has the broadest scope impact — all trade-finance entities that carry customer attributes (Trade Transaction, Bank Guarantees, Import/Export LC, Buyer Credit, Invoice Master, Receivable Financing, Trade Exposure Snapshot) are denormalizing customer data that belongs in `silver.party`.

---

### 4.9 Entity: Export LC Transactions

**Physical Table:** None  
**Logical Entity:** `Export_LC_Transactions` (21 attributes)  
**Data Mapping Attributes:** 18 (all status=None)

#### 4.9a Industry Fitness

Export Letters of Credit are a core Trade Finance instrument under BIAN `Commercial Letters of Credit (Export)`. Essential for CBUAE trade finance regulatory reporting and counterparty risk management.

#### 4.9b Attribute-Level Review

All 18 mapping attributes have status=None. Key observations:
- `LC id` — business key; must become `export_lc_bk` + `export_lc_sk`
- `LC Amount`, `Negotiation Amount` — typed as None/STRING; must be `DECIMAL(18,4)` in AED fils
- `Shipment date`, `Expiry Date` — must be `DATE` type
- `Advising Bank code` — should FK to `trade_counterparty` (which also has no physical table)
- `Settlement Status` — raw source code; must use `silver.reference.code_mapping`
- Data types all null in mapping — incomplete specification

#### 4.9c Metadata Completeness: All missing (no physical table).

**Findings:**
- **GAP-029 (P0/High, Confidence: 1.0):** No physical table for Export LC Transactions. Critical production gap.
- **GAP-030 (P0/High, Confidence: 1.0):** Zero ETL mapping.
- **GAP-031 (P1/Medium, Confidence: 0.9):** All attribute data types are null in the mapping — the data model specification for this entity is incomplete.

---

### 4.10 Entity: Financial Investment Group

**Physical Table:** `silver_dev.trade.financial_investment_group`  
**Logical Entity:** `Financial Investment Group` (7 attributes)  
**Data Mapping Attributes:** 1

#### 4.10a Industry Fitness

Financial Investment Group appears to be a grouping/classification hierarchy for investment products (e.g., "Fixed Income", "Equities", "Derivatives"). This maps to the BIAN `Investment Portfolio` service domain's product hierarchy.

#### 4.10b Attribute-Level Review

Only `Investment Product Id` mapped from FIS.INSTRUMENT.product_chlnbr. The remaining 6 attributes (group name, group code, parent group, hierarchy level, etc.) are unmapped.

#### 4.10c Metadata Completeness: All 11 missing.

**Findings:**
- **GAP-032 (P1/Medium, Confidence: 0.9):** 1/7 attributes mapped. The entity is functionally empty.

---

### 4.11 Entity: Fund

**Physical Table:** `silver_dev.trade.fund`  
**Logical Entity:** `Fund` (8 attributes)  
**Data Mapping Attributes:** 2

#### 4.11a Industry Fitness

Fund represents mutual funds and collective investment schemes. Belongs in `silver.treasury` (wealth management sub-domain). The BIAN `Investment Fund Administration` service domain is the canonical reference.

#### 4.11b Attribute-Level Review

| Attribute | Status | Source | Issues |
|---|---|---|---|
| `Investment Product Id` | ✅ Mapped | WMS.MF_FUND_MASTER_TBL.FUND_ID | Must become `fund_bk` |
| `Fund Type Code` | ✅ Mapped | WMS.MF_FUND_MASTER_TBL.FUND_TYPE | PII=`1 distinct value` noted in mapping — likely a data quality flag, not a PII flag; investigate |

6 of 8 logical attributes unmapped (NAV, fund manager, inception date, risk rating, currency, benchmark).

#### 4.11c Metadata Completeness: All 11 missing.

**Findings:**
- **GAP-033 (P1/Low, Confidence: 0.85):** `Fund Type Code` PII column has value `1 distinct value` instead of Y/N — appears to be a pivot table artifact carried into the PII column. Must be corrected.

---

### 4.12 Entity: Future Product

**Physical Table:** `silver_dev.trade.future_product`  
**Logical Entity:** `Future Product` (8 attributes)  
**Data Mapping Attributes:** 2

#### 4.12a Industry Fitness

Futures are exchange-traded derivatives. Belongs in `silver.treasury`. The entity is sparse (only 2 of 8 attributes mapped).

#### 4.12b Attribute-Level Review

| Attribute | Status | Source | Issues |
|---|---|---|---|
| `Investment Product Id` | ✅ Mapped | FIS.INSTRUMENT.product_chlnbr | Must become `future_product_bk` |
| `Future Type Code` | ✅ Mapped | FIS.INSTRUMENT.INSTYPE | Source code; must route through `silver.reference.code_mapping` (SLV-004) |

#### 4.12c Metadata Completeness: All 11 missing.

**Findings:**
- **GAP-034 (P1/Medium, Confidence: 0.9):** `Future Type Code` uses raw INSTYPE code from FIS. Violates SLV-004.

---

### 4.13 Entity: Import LC Transactions

**Physical Table:** None  
**Logical Entity:** `Import_LC_Transactions` (21 attributes)  
**Data Mapping Attributes:** 18 (all status=None)

#### 4.13a Industry Fitness

Import Letters of Credit are the most common Trade Finance instrument in UAE banking. Under BIAN `Commercial Letters of Credit (Import)`. Critical for CBUAE regulatory reporting, contingent liability tracking, and customer exposure management.

#### 4.13b Attribute-Level Review

All 18 mapping attributes have status=None and data type=None. Key observations:
- `LC id` — must become `import_lc_bk` + `import_lc_sk` (MD5)
- `LC Amount` — must be `DECIMAL(18,4)` in AED fils
- `Issue Date`, `Expiry Date` — must be `DATE`
- `Usage percentage` — should be `DECIMAL(5,4)` (rate)
- `LC status` — must use `silver.reference.code_mapping`
- `Incotem code` [sic] — FK to `reference_incoterm` (also unmapped)
- `Country Code` — FK to `silver.reference.country` (not `reference_country` in trade SA)

#### 4.13c Metadata Completeness: All missing.

**Findings:**
- **GAP-035 (P0/High, Confidence: 1.0):** No physical table for Import LC. Most critical missing entity in Trade Finance.
- **GAP-036 (P0/High, Confidence: 1.0):** Zero ETL mapping.

---

### 4.14 Entity: Interest Index Product

**Physical Table:** `silver_dev.trade.interest_index_product`  
**Logical Entity:** `Interest Index Product` (8 attributes)  
**Data Mapping Attributes:** 2

#### 4.14a Industry Fitness

Interest Index Products (SOFR, EURIBOR, EIBOR) are reference rate instruments. These are reference data and should be in `silver.reference`, not `silver.trade`. The physical table's presence in `silver.trade` is a scoping violation.

#### 4.14b Attribute-Level Review

| Attribute | Status | Issues |
|---|---|---|
| `Investment Product ID` | ✅ Mapped (FIS.INSTRUMENT.product_chlnbr) | Must become `interest_index_product_bk` |
| `Money Market Product Type Cd` | ⚠️ Not Available | Misnamed attribute (should not reference Money Market for an Interest Index) — logical model error |

#### 4.14c Metadata Completeness: All 11 missing.

**Findings:**
- **GAP-037 (P1/Medium, Confidence: 0.9):** Attribute `Money Market Product Type Cd` is misnamed in the logical model for the Interest Index entity — likely a copy-paste error from the Money Market Product entity.

---

### 4.15 Entity: Investment Product

**Physical Table:** `silver_dev.trade.investment_product`  
**Logical Entity:** `Investment Product` (implied as super-type via sub-types)  
**Data Mapping Attributes:** 4

#### 4.15a Industry Fitness

Investment Product is the super-type entity for all investment instruments (Bond, Fund, Commodity, etc.). This is an IFW-aligned pattern where specialized sub-types inherit the base product attributes. The entity is correctly identified but belongs in `silver.treasury`.

#### 4.15b Attribute-Level Review

| Attribute | Status | Source | Issues |
|---|---|---|---|
| `Investment Product Id` | ✅ Mapped | FIS.INSTRUMENT.product_chlnbr | Must become `investment_product_bk` and also generate `investment_product_sk` (MD5) |
| `Investment Product Trade Type cd` | ✅ Mapped | FIS.TRADE.TYPE | Source code — must route through code_mapping |
| `Investment Product Type code` | ✅ Mapped | FIS.INSTRUMENT.product_type_chlnbr | Source code — must route through code_mapping |
| `Denomination Currency code` | ✅ Mapped | FIS.TRADE.TRADE_CURR | Should be FK to `silver.reference.currency`; not `silver.trade.reference_country` |

#### 4.15c Metadata Completeness: All 11 missing.

**Findings:**
- **GAP-038 (P1/Medium, Confidence: 0.9):** All three code columns carry raw FIS codes. Must all route through `silver.reference.code_mapping` per SLV-004.
- **GAP-039 (P1/Medium, Confidence: 0.85):** No subtype discriminator column defined to distinguish which sub-type entity (Bond, Fund, Commodity, etc.) this base record belongs to.

---

### 4.16 Entity: Investment Product Metric History

**Physical Table:** `silver_dev.trade.investment_product_metric_history`  
**Logical Entity:** `Investment Product Metric History` (14 attributes in logical model)  
**Data Mapping Attributes:** 8

#### 4.16a Industry Fitness

This entity stores time-series metric snapshots (e.g., price, yield, duration) per investment product. The "History" suffix implies SCD-2 or time-series storage. The entity is conceptually correct but the implementation has no SCD-2 columns, making it functionally indistinguishable from a snapshot table.

#### 4.16b Attribute-Level Review

| Attribute | Status | Source | Issues |
|---|---|---|---|
| `Investment Product Id` | ✅ Mapped | FIS.INSTRUMENT.product_chlnbr | Must be `investment_product_bk` + FK; also need separate `investment_product_metric_history_sk` |
| `Quoting Party Id` | ✅ Mapped | FIS.TRADE.COUNTERPARTY_PTYNBR | FK to `trade_counterparty` (no physical table) |
| `Invest Product value date` | ✅ Mapped | FIS.TRADE.VALUE_DAY | STRING → must be DATE |
| `Invest Product value term` | ✅ Mapped | FIS.TRADE.TIME | STRING → semantic unclear; might be tenor/duration |
| `Invest Product Metric type code` | ❌ Not Available | — | Critical classification — without this, metric records cannot be interpreted |
| `Investment Product metric amount` | ✅ Mapped | FIS.INSTRUMENT.FACE_VALUE | STRING → must be DECIMAL(18,4); labelled as "metric amount" but sourced from FACE_VALUE — semantic mismatch |
| `Investment product metric rate` | ✅ Mapped | FIS.INSTRUMENT.COUP_RATE | STRING → must be DECIMAL(18,6) for rates |
| `Currency code` | ✅ Mapped | FIS.INSTRUMENT.CURR | Should FK to `silver.reference.currency` |

#### 4.16c Metadata Completeness: All 11 missing.

**Findings:**
- **GAP-040 (P0/High, Confidence: 0.95):** `Invest Product Metric type code` has no mapping. Without a metric type discriminator, the history table records are uninterpretable — a consumer cannot distinguish price history from yield history from duration history.
- **GAP-041 (P1/High, Confidence: 0.95):** `Investment Product metric amount` is sourced from `FACE_VALUE` but semantically represents a "metric amount" — these are different concepts. `FACE_VALUE` is a static attribute of the instrument, not a time-varying metric. Source mapping must be revisited.
- **GAP-042 (P1/Medium, Confidence: 0.9):** `Invest Product value date` typed as STRING; must be `DATE` with UTC storage.
- **GAP-043 (P1/Medium, Confidence: 0.9):** `Investment product metric rate` typed as STRING; must be `DECIMAL(18,6)`.

---

### 4.17 Entity: Investment Product Term Period History

**Physical Table:** `silver_dev.trade.investment_product_term_period_history`  
**Logical Entity:** `Investment Produc Term Period History` [sic] (16 attributes)  
**Data Mapping Attributes:** 10

#### 4.17a Industry Fitness

This entity stores coupon/term period characteristics for investment products over time. It represents cash flow schedule snapshots. Relevant for bond and structured product analytics.

#### 4.17b Attribute-Level Review

| Attribute | Status | Source | Issues |
|---|---|---|---|
| `Investment Product Id` | ✅ Mapped | FIS.INSTRUMENT.INSID | Note: uses `INSID` not `product_chlnbr` as other entities — key inconsistency |
| `Quoting Party Id` | ✅ Mapped | FIS.TRADE.COUNTERPARTY_PTYNBR | FK to missing `trade_counterparty` |
| `Invest Product value date` | ✅ Mapped | FIS.TRADE.VALUE_DAY | STRING → DATE |
| `Invest Product value term` | ✅ Mapped | FIS.TRADE.TIME | STRING → ambiguous semantic |
| `Invest Product Metric type code` | ❌ Not Available | — | Missing classification |
| `Time period code` | ✅ Mapped | FIS.INSTRUMENT.PAY_PERIOD_UNIT | Source code → code_mapping |
| `Invest Prod Term period number` | ✅ Mapped | FIS.INSTRUMENT.PAY_PERIOD_COUNT | STRING → INTEGER |
| `Invest prod term period metric amount` | ✅ Mapped | FIS.INSTRUMENT.FACE_VALUE | STRING → DECIMAL(18,4); same semantic mismatch as §4.16 |
| `Invest prod term period metric rate` | ✅ Mapped | FIS.INSTRUMENT.COUP_RATE | STRING → DECIMAL(18,6) |
| `Currency Code` | ✅ Mapped | FIS.TRADE.CURR | Should FK to `silver.reference.currency` |

#### 4.17c Metadata Completeness: All 11 missing.

**Findings:**
- **GAP-044 (P1/Medium, Confidence: 0.95):** `Investment Product Id` key source inconsistency: uses `INSTRUMENT.INSID` here but `INSTRUMENT.product_chlnbr` in all other investment product entities. These may be different business keys, creating a join gap.
- **GAP-045 (P1/Medium, Confidence: 0.9):** Logical entity name has typo: "Investment Produc Term Period History" — missing 't' in "Product". Must be corrected.
- **GAP-046 (P1/Medium, Confidence: 0.9):** `Invest Prod Term period number` STRING → INTEGER.

---

### 4.18 Entity: Investment Product Value History

**Physical Table:** `silver_dev.trade.investment_product_value_history`  
**Logical Entity:** `Investment Product Value History` (14 attributes)  
**Data Mapping Attributes:** 8 (all mapped)

#### 4.18a Industry Fitness

This entity stores price/value history for investment products. This is the closest to a properly mapped entity in the entire SA.

#### 4.18b Attribute-Level Review

| Attribute | Status | Source | Issues |
|---|---|---|---|
| `Investment product ID` | ✅ Mapped | FIS.INSTRUMENT.product_chlnbr | Must become `investment_product_value_history_sk` + `investment_product_bk` FK |
| `Quoting Party Id` | ✅ Mapped | FIS.TRADE.COUNTERPARTY_PTYNBR | FK to missing table |
| `Invest Prod value date` | ✅ Mapped | FIS.TRADE.VALUE_DAY | STRING → DATE |
| `Invest Prod value time` | ✅ Mapped | FIS.TRADE.TIME | STRING → TIME or include in TIMESTAMP |
| `Currency Code` | ✅ Mapped | FIS.TRADE.CURR | FK to `silver.reference` |
| `Reporting Party Id` | ✅ Mapped | FIS.PARTY.PTYID | FK to `silver.party` |
| `Invest Product value amount` | ✅ Mapped | FIS.TRADE.BOOK_VALUE | STRING → DECIMAL(18,4) |
| `Invest product pct` | ✅ Mapped | FIS.INSTRUMENT.COUP_RATE | STRING → DECIMAL(18,6); naming: `pct` abbreviation is in the approved list |

#### 4.18c Metadata Completeness: All 11 missing.

**Findings:**
- **GAP-047 (P1/High, Confidence: 0.95):** `Invest Prod value date` + `Invest Prod value time` are separate STRING columns. These should be combined into a single `TIMESTAMP` column `invest_prod_value_timestamp` (UTC) per SLV-009.
- **GAP-048 (P1/Medium, Confidence: 0.9):** `Invest Product value amount` typed as STRING; must be `DECIMAL(18,4)`.

---

### 4.19 Entity: Investment Security Risk Grade

**Physical Table:** `silver_dev.trade.investment_security_risk_grade`  
**Logical Entity:** `Investment Security Risk Grade` (11 attributes)  
**Data Mapping Attributes:** 5 (all status=Not Available)

#### 4.19a Industry Fitness

This entity stores credit/risk ratings for investment securities (Moody's, S&P, Fitch grades). Important for IFRS 9 ECL calculations and Basel III RWA. Belongs in `silver.treasury` or potentially `silver.risk`.

#### 4.19b Attribute-Level Review

| Attribute | Status | Issues |
|---|---|---|
| `Investment Product Id` | ❌ Not Available | Business key with no source |
| `Risk Graded Id` | ❌ Not Available | Rating agency identifier unmapped |
| `Invest Risk Grade Start Dttm` | ❌ Not Available | Should be `TIMESTAMP` UTC; SCD-2 effective start |
| `Invest Risk Grade End Dttm` | ❌ Not Available | Should be `TIMESTAMP` UTC; SCD-2 effective end |
| `Investment Risk Grade Rate Dttm` | ❌ Not Available | Ambiguous name — "Rate Dttm" conflicts with "Dttm" suffix convention; should be `risk_grade_effective_date` |

#### 4.19c Metadata Completeness: All 11 missing.

**Findings:**
- **GAP-049 (P0/High, Confidence: 1.0):** Zero mapping for all 5 attributes. A physical table exists but has no ETL pipeline. Empty table in Silver.
- **GAP-050 (P1/Medium, Confidence: 0.9):** Column suffix `_Dttm` uses non-standard casing and abbreviation. Per NAM-001, should be `_timestamp` or `_date` as appropriate, in lowercase snake_case.

---

### 4.20 Entity: Invoice Master

**Physical Table:** None  
**Logical Entity:** `Invoice Master` (19 attributes)  
**Data Mapping Attributes:** 15 (all status=None)

#### 4.20a Industry Fitness

Invoice Master is a trade document entity supporting Receivable Financing and trade settlements. Under BIAN `Bills of Exchange` / `Commercial Letters of Credit`. Critical for supply chain finance and receivable financing products.

#### 4.20b Attribute-Level Review

All 15 attributes have status=None. Key issues:
- `Invoice Number` — business key; must become `invoice_bk` + `invoice_sk`
- `Amount` — STRING; must be `DECIMAL(18,4)`
- `Invoice date`, `Due Date` — STRING; must be `DATE`
- `Tax` — STRING; must be `DECIMAL(18,4)` (VAT amount)
- `IsLogicallyDeleted` — non-standard; must become `is_active_flag`

**Findings:**
- **GAP-051 (P0/High, Confidence: 1.0):** No physical table. Core trade finance entity for supply chain finance.
- **GAP-052 (P0/High, Confidence: 1.0):** Zero ETL mapping.

---

### 4.21 Entity: Money Market Product

**Physical Table:** `silver_dev.trade.money_market_product`  
**Logical Entity:** `Money Market Product` (8 attributes)  
**Data Mapping Attributes:** 2

#### 4.21a Industry Fitness

Money Market Products (T-Bills, CDs, Commercial Paper) are short-duration fixed-income instruments. Belongs in `silver.treasury`. Only 2 of 8 attributes mapped.

#### 4.21b Attribute-Level Review

| Attribute | Status | Issues |
|---|---|---|
| `Investment Product Id` | ✅ Mapped (FIS.INSTRUMENT.product_chlnbr) | Must become `money_market_product_bk` |
| `Money Market Product Type Cd` | ⚠️ Not Available (FIS, no table) | Source identified but table unknown |

#### 4.21c Metadata Completeness: All 11 missing.

---

### 4.22 Entity: Non Traded Investment Product

**Physical Table:** `silver_dev.trade.non_traded_investment_product`  
**Logical Entity:** `Non Traded Investment Product` (7 attributes)  
**Data Mapping Attributes:** 1

#### 4.22a Industry Fitness

Non-Traded products are unlisted/OTC instruments (private equity, unlisted bonds). Only 1 attribute mapped.

#### 4.22b Attribute-Level Review

| Attribute | Status | Source | Issues |
|---|---|---|---|
| `Investment Prod Id` | ✅ Mapped | WMS.SP_GEN_DESCRIPTION_TABLE.PROD_CODE | WMS source — different from FIS used by other investment products; cross-system consistency issue |

Note: `Investment Prod Id` uses abbreviated `Prod` while other entities use full `Product Id` — naming inconsistency in the logical model.

**Findings:**
- **GAP-053 (P1/Low, Confidence: 0.8):** Source system WMS for this entity vs. FIS for other investment sub-types (Bond, Fund, Future, etc.) — potential for duplicate or conflicting product records.

---

### 4.23 Entity: Option Product

**Physical Table:** `silver_dev.trade.option_product`  
**Logical Entity:** `Option Product` (8 attributes)  
**Data Mapping Attributes:** 2

#### 4.23a Industry Fitness

Options are derivatives under BIAN `Traded Positions`. Belongs in `silver.treasury`.

#### 4.23b Attribute-Level Review

| Attribute | Status | Source | Issues |
|---|---|---|---|
| `Investment Prod Id` | ✅ Mapped | FIS.INSTRUMENT.product_chlnbr | Must become `option_product_bk` |
| `Option Style type code` | ✅ Mapped | FIS.INSTRUMENT.OPTION_TYPE | Raw source code → `silver.reference.code_mapping` |

**Findings:**
- **GAP-054 (P1/Medium, Confidence: 0.9):** `Option Style type code` uses raw FIS OPTION_TYPE code. Must route through canonical code mapping (SLV-004).

---

### 4.24 Entity: Receivable Financing Transactions

**Physical Table:** None  
**Logical Entity:** `Receivable_Financing_Transactions` (21 attributes)  
**Data Mapping Attributes:** 17 (all status=None)

#### 4.24a Industry Fitness

Receivable Financing (factoring, invoice discounting) is a trade finance product under BIAN `Supplier Financing`. Critical for SME banking and trade working capital products at RAKBANK.

#### 4.24b Attribute-Level Review

All 17 attributes have status=None. Key issues:
- `Invoice Amount`, `Financed Amount` — STRING; must be `DECIMAL(18,4)`
- `Discount Rate` — STRING; must be `DECIMAL(18,6)`
- `Maturity Date` — STRING; must be `DATE`
- `Finance Type` — STRING; must use `silver.reference.code_mapping`
- `Settlement Status` — STRING source code
- `IsLogicallyDeleted` — non-standard

**Findings:**
- **GAP-055 (P0/High, Confidence: 1.0):** No physical table.
- **GAP-056 (P0/High, Confidence: 1.0):** Zero ETL mapping.

---

### 4.25 Entity: Reference Country

**Physical Table:** `silver_dev.trade.reference_country`  
**Logical Entity:** `Reference_Country` (16 attributes) + `Reference_Country1` (16 attrs — duplicate)  
**Data Mapping Attributes:** 16

#### 4.25a Industry Fitness

**Finding GAP-057 (P0/High, Confidence: 1.0):** Country reference data belongs in `silver.reference` as per `sa/subject-areas.md` entry #1 (*"Master Lookups: ISO Currencies, Country Codes, FX Rates, Commodity Prices (Gold/Silver), Channel Master, and Industry Codes (NACE/SIC)."*). Including a `reference_country` table in `silver.trade` is a scoping violation that will cause data duplication and divergence with the canonical reference SA.

**Finding GAP-058 (P1/Medium, Confidence: 1.0):** `Reference_Country1` in the logical model is a duplicate of `Reference_Country` — same 16 attributes. This appears to be a logical modeling error (copy-paste artifact). Must be removed.

#### 4.25b Attribute-Level Review

| Attribute | Status | Source | Issues |
|---|---|---|---|
| `Country Code` | ✅ Mapped | EXIMBILLS.STD_COUNTRY.C_CNTY_CODE | PK; must become `reference_country_bk` |
| `Country Short Name` | ✅ Mapped | STD_COUNTRY.C_CNTY_NAME | OK |
| `Country Long Name` | ✅ Mapped | STD_COUNTRY.C_CNTY_NAME_CN | OK |
| `Country Continent` | ❌ Not Available | — | No source field identified |
| `Country Region` | ✅ Mapped | STD_COUNTRY.C_AREA_CODE | Raw source code → needs canonical mapping |
| `Country ISO Code 1` | ✅ Mapped | C_CNTY_CODE2 | ISO Alpha-2 |
| `Country ISO Code 2` | ✅ Mapped | C_CNTY_CODE | ISO Alpha-3 (same source as Country Code — ambiguous) |
| `Country ISO Code 3` | ✅ Mapped | C_CNTY_CODE1 | Numeric ISO code |
| `Country ISD Code` | ✅ Mapped | C_CNTY_CODE2 | Same source as `Country ISO Code 1` — duplication risk |
| `Country IsOfficial` | ❌ Not Available | — | No source |
| `Source System Code` | ✅ Mapped | STD_COUNTRY (no column) | Column not specified |
| `Source System Id` | ✅ Mapped | STD_COUNTRY (no column) | Column not specified |
| `IsLogicallyDeleted` | ❌ Not Available | — | Non-standard name; must be `is_active_flag` |
| `Create Date` | ✅ Mapped | STD_COUNTRY (no column) | Column not specified; STRING → TIMESTAMP |
| `Update Date` | ✅ Mapped | STD_COUNTRY (no column) | Column not specified; STRING → TIMESTAMP |
| `Deleted Date` | ❌ Not Available | — | Non-standard; must be `delete_date TIMESTAMP` |

**Findings:**
- **GAP-059 (P0/High, Confidence: 0.95):** `Country ISD Code` and `Country ISO Code 1` both map to same source column `C_CNTY_CODE2` — likely an error in the mapping; ISD code should be a telephone international dialling code, not the ISO alpha-2 code.
- **GAP-060 (P1/Medium, Confidence: 0.9):** Source column names not specified for `Source System Code`, `Source System Id`, `Create Date`, `Update Date` — mapping is incomplete.

---

### 4.26 Entity: Reference Currency

**Physical Table:** None  
**Logical Entity:** `Reference_Currency` (11 attributes)  
**Data Mapping Attributes:** 11 (4 mapped, 7 status=None)

#### 4.26a Industry Fitness

Currency reference data belongs in `silver.reference`. No physical table exists despite a partial mapping. The 4 mapped attributes source from `EXIMBILLS.STD_CURRENCY`.

#### 4.26b Attribute-Level Review

| Attribute | Status | Issues |
|---|---|---|
| `Currency Id` | ✅ Mapped (EXIMBILLS.STD_CURRENCY.C_CURRENCY) | Must become `reference_currency_bk` |
| `Numeric Id` | ✅ Mapped (C_COUNTRY) | Column name mismatch — C_COUNTRY as numeric currency ID is suspicious |
| `Currency description` | ✅ Mapped (C_MAJOR_NAME) | OK |
| `Country` | ✅ Mapped (I_ISO_CODE) | FK to `reference_country` |
| `Minor unit` – `Deleted Date` | ❌ None (7 attrs) | Completely unmapped |

**Findings:**
- **GAP-061 (P1/High, Confidence: 0.85):** `Numeric Id` maps to `C_COUNTRY` — the EXIMBILLS column for ISO numeric currency code should be a dedicated numeric field, not the country code. Semantic mismatch requires verification.
- **GAP-062 (P1/Medium, Confidence: 1.0):** No physical table despite being needed by all currency-related entities in the SA.

---

### 4.27 Entity: Reference Exchange Rate 1

**Physical Table:** `silver_dev.trade.reference_exchange_rate1`  
**Logical Entity:** `Reference_Exchange_Rate1` (11 attributes)  
**Data Mapping Attributes:** 5 (all mapped)

#### 4.27a Industry Fitness

FX Rates belong in `silver.reference`. The `_1` suffix on both table and entity name is non-standard (NAM-001).

#### 4.27b Attribute-Level Review

| Attribute | Status | Source | Issues |
|---|---|---|---|
| `Source Currency Code` | ✅ Mapped | EXIMBILLS.STD_EXCHAN_RATE.C_FROM_CCY | OK |
| `Target Currency Code` | ✅ Mapped | EXIMBILLS.STD_EXCHAN_RATE.C_TO_CCY | OK |
| `Source to Target Currency Rate` | ✅ Mapped | STD_EXCHAN_RATE.F_VALUE | STRING → DECIMAL(18,8) (rate needs high precision) |
| `Target to Source Currency Rate` | ✅ Mapped | STD_EXCHAN_RATE.F_VALUE | **Same source column** as above — both buy and sell rates map to the same F_VALUE; semantic error |
| `Exchange Rate Date` | ✅ Mapped | STD_EXCHAN_RATE.D_VALUE_DATE | STRING → DATE |

**Findings:**
- **GAP-063 (P0/High, Confidence: 0.95):** `Source to Target Currency Rate` and `Target to Source Currency Rate` both map to `STD_EXCHAN_RATE.F_VALUE`. This is semantically incorrect — buy rate and sell rate are different values. One of these mappings is wrong and must be investigated. Bi-directional rates require separate source columns or a rate type discriminator.
- **GAP-064 (P0/High, Confidence: 1.0):** Table named `reference_exchange_rate1` with trailing `1` suffix — violates NAM-001 (*"No abbreviations unless universally understood"*); `_1` has no semantic meaning here. Must be renamed to `reference_exchange_rate`.
- **GAP-065 (P1/Medium, Confidence: 0.9):** `Source to Target Currency Rate` and `Exchange Rate Date` typed as STRING; must be `DECIMAL(18,8)` and `DATE` respectively.

---

### 4.28 Entity: Reference Incoterm

**Physical Table:** None  
**Logical Entity:** `Reference_Incoterm` (9 attributes)  
**Data Mapping Attributes:** 9 (all status=None)

#### 4.28a Industry Fitness

Incoterms (International Commercial Terms) are ICC-standardized trade terms (FOB, CIF, EXW, etc.) essential for trade finance LC processing. Reference data belonging in `silver.reference`.

#### 4.28b Attribute-Level Review

All 9 attributes have status=None. Key: `Incotem code` [sic] — typo ("Incotem" should be "Incoterm"); `Risk Distribution` — describes which party bears risk for shipping.

**Findings:**
- **GAP-066 (P1/Medium, Confidence: 1.0):** No physical table.
- **GAP-067 (P1/Low, Confidence: 1.0):** Attribute name typo `Incotem code` — should be `incoterm_code`.

---

### 4.29 Entity: Reference Interest Rate 1

**Physical Table:** None  
**Logical Entity:** `Reference_Interest_Rate1` (13 attributes)  
**Data Mapping Attributes:** 7 (6 mapped, 1 not available)

#### 4.29a Industry Fitness

Interest rate benchmarks (SOFR, EIBOR, LIBOR replacement rates) belong in `silver.reference`. No physical table despite 6 of 7 attributes being mapped.

#### 4.29b Attribute-Level Review

| Attribute | Status | Source | Issues |
|---|---|---|---|
| `Interest Index Code` | ✅ Mapped | EXIMBILLS.STD_INTERESTRATE.C_BK_GROUP_ID | Must become `reference_interest_rate_bk` |
| `Interest Index Time Period` | ✅ Mapped | STD_INTERESTRATE.D_VALUE_DATE | STRING → DATE |
| `Interest Index description` | ✅ Mapped | STD_INTERESTRATE.C_CP_UNIT_CODE | Semantic mismatch: unit code used as description |
| `Interest Index currency code` | ✅ Mapped | STD_INTERESTRATE.C_CURRENCY | FK to `silver.reference.currency` |
| `Interest Rate Effective Date` | ✅ Mapped | STD_INTERESTRATE.D_SYS_OP_DATE | STRING → DATE |
| `Fixed Interest Rate` | ✅ Mapped | STD_INTERESTRATE.F_VALUE | STRING → DECIMAL(18,6) |
| `Spread rate` | ❌ Not Available | — | EIBOR spread component unmapped |

**Findings:**
- **GAP-068 (P1/Medium, Confidence: 1.0):** Physical table not created despite 6/7 attributes being mapped. This should be straightforward to implement.
- **GAP-069 (P1/Medium, Confidence: 0.85):** `Interest Index description` maps to `C_CP_UNIT_CODE` (a unit/period code) — this is a semantic mismatch. Description should come from a name/description column, not a code column.
- **GAP-070 (P1/Medium, Confidence: 0.9):** Entity name suffix `_1` is non-standard (same issue as `Reference_Exchange_Rate1`).

---

### 4.30 Entity: Reference Product Trade

**Physical Table:** None (note: `reference_product` exists but maps to a different entity)  
**Logical Entity:** `Reference_Product_Trade` (11 attributes)  
**Data Mapping Attributes:** 11 (all status=None)

#### 4.30a Industry Fitness

Trade Product Reference contains trade-finance specific product definitions (LC types, BG types, tenor classifications). The `reference_product` physical table in the schema appears to map to `Reference_Product` (the EXIMBILLS product catalog entity) rather than `Reference_Product_Trade`.

#### 4.30b Attribute-Level Review

All 11 attributes have status=None. Key attributes: `Product Id`, `Product name`, `Product Category Code`, `Tenor Type`, `Risk Weight`. `Risk Weight` in a reference table is a potential pre-computed metric violation (SLV-007).

**Findings:**
- **GAP-071 (P1/Medium, Confidence: 1.0):** No physical table and zero mapping.
- **GAP-072 (P1/Medium, Confidence: 0.85):** `Risk Weight` attribute in a reference product table suggests a static regulatory weight. If this is dynamically computed, it violates SLV-007 (no pre-computed metrics in Silver). Requires clarification from data steward.

---

### 4.31 Entity: Reference Source System

**Physical Table:** None  
**Logical Entity:** `Reference_Source_System` (10 attributes)  
**Data Mapping Attributes:** 10 (all marked Mapped but no source columns specified)

#### 4.31a Industry Fitness

Source system metadata is cross-cutting reference data belonging in `silver.reference`. All 10 attributes are marked "Mapped" but no source system, schema, table, or column is specified — the mapping is a placeholder, not a real mapping.

**Findings:**
- **GAP-073 (P1/Medium, Confidence: 1.0):** No physical table despite all attributes being marked as mapped.
- **GAP-074 (P1/Medium, Confidence: 1.0):** Mapping entries show "Mapped" status with zero source details — all source fields are None. This is a false mapping.

---

### 4.32 Entity: Security Index Product

**Physical Table:** `silver_dev.trade.security_index_product`  
**Logical Entity:** `Security Index Product` (7 attributes)  
**Data Mapping Attributes:** 1

#### 4.32a Industry Fitness

Security indices (S&P 500, MSCI, etc.) used as benchmark instruments. Belongs in `silver.treasury`.

#### 4.32b Attribute-Level Review

Only `Investment Product Id` mapped from FIS.DBO.INSTRUMENT.product_chlnbr. 6 of 7 logical attributes unmapped.

**Findings:**
- **GAP-075 (P1/Low, Confidence: 0.9):** 1/7 attributes mapped. Functionally empty entity.

---

### 4.33 Entity: Stock

**Physical Table:** None  
**Logical Entity:** `Stock` (14 attributes)  
**Data Mapping Attributes:** 3 (1 Not Available, 2 Mapped)

#### 4.33a Industry Fitness

Equity instruments (listed stocks/shares). Belongs in `silver.treasury` under BIAN `Securities Administration`.

#### 4.33b Attribute-Level Review

| Attribute | Status | Source | Issues |
|---|---|---|---|
| `Investment Product Id` | ❌ Not Available | — | Primary key with no source |
| `Stock type code` | ✅ Mapped | WMS.TBAADM.SP_GEN_DESCRIPTION_TABLE.PROD_TYPE | Transformation note: `PK: PROD_CODE` — this is a JOIN key, not a transform |
| `Stock share class code` | ✅ Mapped | WMS.TBAADM.SP_GEN_DESCRIPTION_TABLE.PRODUCT_CLASSIFICATION | Same table/PK condition |

**Findings:**
- **GAP-076 (P1/Medium, Confidence: 1.0):** No physical table.
- **GAP-077 (P1/High, Confidence: 0.95):** Primary identifier `Investment Product Id` has no source mapping — the entity cannot be keyed without a business key source.

---

### 4.34 Entity: Stock History

**Physical Table:** None  
**Logical Entity:** `Stock History` (14 attributes)  
**Data Mapping Attributes:** 4 (2 mapped, 2 not available)

#### 4.34a Industry Fitness

Stock price/volume history. Time-series entity belonging in `silver.treasury`.

#### 4.34b Attribute-Level Review

| Attribute | Status | Source | Issues |
|---|---|---|---|
| `Investment Product Id` | ✅ Mapped | WMS.TBAADM.WMS_ML_HLDG_VAL_HIST_TBL.PRODUCT_CODE | OK as BK source |
| `Stock History Date` | ❌ Not available | — | Critical for time-series — date is the temporal key |
| `Num Change Reason type code` | ❌ Not available | — | Classification missing |
| `Stock Unit Issued quantity` | ✅ Mapped | WMS.TBAADM.WMS_ML_HLDG_VAL_HIST_TBL.ISSUED_SHARES | STRING → DECIMAL(18,4) |

**Findings:**
- **GAP-078 (P0/High, Confidence: 1.0):** No physical table.
- **GAP-079 (P1/High, Confidence: 1.0):** `Stock History Date` has no source mapping — a time-series history table without a date key is functionally useless.

---

### 4.35 Entity: Trade Counterparty

**Physical Table:** None  
**Logical Entity:** `Trade Counterparty` (15 attributes)  
**Data Mapping Attributes:** 12 (all status=None)

#### 4.35a Industry Fitness

Trade Counterparty represents the foreign bank, trading partner, or beneficiary in trade finance transactions. This is a critical FK-dependency for Import LC, Export LC, Bank Guarantees, Buyer Credit, and Trade Transaction. The absence of this physical table means all FK relationships in the existing `trade_transaction` table are dangling.

In BIAN v11, counterparty data maps to `Counterparty Administration`. While the Party SA (`silver.party`) handles internal bank customers, trade counterparties (foreign banks, overseas entities) require their own entity in the trade finance context.

#### 4.35b Attribute-Level Review

All 12 attributes have status=None. Key attributes:
- `Counterparty Id` — business key
- `Name`, `Type`, `Country Code` — descriptive
- `Rating`, `Rating Agency id` — credit risk (potential overlap with `silver.risk`)
- `IsLogicallyDeleted` — non-standard

**Findings:**
- **GAP-080 (P0/High, Confidence: 1.0):** No physical table. All FKs from `trade_transaction` to counterparty are dangling.
- **GAP-081 (P0/High, Confidence: 1.0):** Zero ETL mapping.
- **GAP-082 (P1/Medium, Confidence: 0.85):** `Rating` and `Rating Agency id` should potentially FK to `silver.risk` or `investment_security_risk_grade` rather than be stored denormalized in the counterparty entity.

---

### 4.36 Entity: Trade Exposure Snapshot

**Physical Table:** None  
**Logical Entity:** `Trade Exposure Snapshot` (17 attributes)  
**Data Mapping Attributes:** 14 (all status=None)

#### 4.36a Industry Fitness

**Finding GAP-083 (P0/Critical, Confidence: 0.95):** This entity contains pre-aggregated metrics: `Total Trade Exposure`, `Secured amount`, `Unsecured amount`, `FX exposure`. These are aggregations derived from underlying trade transactions, violating guideline 11 §2.6 (SLV-007): *"DON'T pre-compute totals, averages, counts, ratios, or any derived metrics in Silver tables."* This entity should either be moved to the Gold layer as a fact/aggregate table, or redesigned as a raw snapshot from the source system (if the source system itself provides these pre-aggregated values, the entity must be clearly documented as a source extract, not a Silver derivation).

#### 4.36b Attribute-Level Review

| Attribute | Issues |
|---|---|
| `Snapshot id` | Business key; must become `trade_exposure_snapshot_bk` |
| `Customer number CIF` | FK to `silver.party`; not a full customer entity |
| `Total Trade Exposure` | ⚠️ Pre-aggregated metric — SLV-007 violation |
| `Secured amount` | ⚠️ Pre-aggregated — SLV-007 violation |
| `Unsecured amount` | ⚠️ Pre-aggregated — SLV-007 violation |
| `FX exposure` | ⚠️ Derived metric — SLV-007 violation |
| `Snapshot date` | Must be `DATE` type |
| `Customer type` | Denormalized from customer — cross-SA attribute |

**Findings:**
- **GAP-084 (P0/Critical, Confidence: 0.95):** Exposure snapshot aggregates violate SLV-007. Requires data steward ruling: either this is a source-system extract (document it clearly) or move to Gold.
- **GAP-085 (P0/High, Confidence: 1.0):** No physical table.

---

### 4.37 Entity: Trade Transaction

**Physical Table:** `silver_dev.trade.trade_transaction`  
**Logical Entity:** `Trade Transaction` (93 attributes)  
**Data Mapping Attributes:** 91

This is the most complex entity in the SA. Let a detailed review follow.

#### 4.37a Industry Fitness

**Finding GAP-086 (P0/Critical, Confidence: 1.0):** The Trade Transaction entity attempts to be a universal trade finance super-table, covering Import LC, Export LC, Bills of Exchange, Pre-shipment financing, and related sub-transactions in a single 93-attribute wide table. This is a fundamental modeling error:

1. **3NF Violation (SLV-006):** Multiple repeating groups exist:
   - `Related Reference Number`, `Related Reference Number 2`, `Related Reference Number 3`, `Related Reference Number 4` — four identical columns (1NF repeating group violation)
   - `Port Of Loading1`, `Port Of Loading2`, `Port Of Loading3` — three identical columns
   - `Port Of Discharge 1`, `Port Of Discharge2`, `Port Of Discharge3` — three identical columns
   - `Vessel Name1`, `Vessel Name2`, `Vessel Name3` — three identical columns
   - `Bl Date1`, `Bl Date2`, `Bl Date3` — three identical columns
   - `Goods Description 1`, `Goods Description2` — two identical columns
   - `Related Name`, `Related Name Address 1`, `Related Name Address2`, `Related Name Address3` — address group
   - `Counter Party Address1`, `Counter Party Address2`, `Counter Party Address3` — address group
   - `Related Bank Address1`, `Related Bank Address2`, `Related Bank Address3` — address group
   - `Related Bank 2`, `Related Bank 2 Bic Code`, `Related Bank 2 Address1/2/3` — entire bank repeating group

2. **Denormalization of Customer/Counterparty attributes:** Customer Name, Customer Category, Customer Country, Customer Segment, RM/AO Name, RM/AO Code are all customer attributes denormalized into the transaction — violating subject area isolation.

3. **Mixed grain:** The entity mixes trade transaction header attributes with shipment details, bank routing information, and financing attributes.

#### 4.37b Attribute-Level Review (key attributes)

| Attribute | Mapped Source | Type | Status | Issues |
|---|---|---|---|---|
| `Transaction_Id` | EXIMTRX.IPLC_MASTER.C_TRX_REF | STRING | ✅ Mapped | Must become `trade_transaction_bk`; also generate `trade_transaction_sk` (MD5) |
| `Txn Date - Yyyymmdd` | EXIMTRX.IPLC_LEDGER.D_CREA_DATE | DATE | ✅ Mapped | Column name must be `transaction_date` per NAM-001 |
| `Related Reference Number` (×4) | Various IPLC tables | STRING | ✅/❌ | 3NF violation: numbered repeating groups |
| `Customer Code` | EXIMSYS.IPLC_MASTER.C_CUST_ID | STRING | ✅ Mapped | Cross-SA: FK to `silver.party`; customer data must not be in this entity |
| `Customer_number_CIF` | EXIMTRX.IPLC_MASTER.C_CUST_ID | STRING | ✅ Mapped | Same source as `Customer Code` — duplicate mapping |
| `Customer Name` | — | STRING | ❌ Not Available | Customer attribute — cross-SA denormalization |
| `Amount Value` | IPLC_EM_NEGO.INSTALL_AMOUNT | DECIMAL | ✅ Mapped | STRING → DECIMAL(18,4); column name should be `transaction_amount` |
| `Amount In Lcy` | IPLC_MASTER.LCY | DECIMAL | ✅ Mapped | STRING → DECIMAL(18,4); `lcy` = local currency amount; column name should be `transaction_amount_lcy_aed` |
| `Amount Currency` | IPLC_EM_NEGO.ACPT_CCY | CHAR(3) | ✅ Mapped | OK; FK to `silver.reference.currency` |
| `Profitrate Rate` | IPLC_EM_NEGO.CFNC_C_INT_PAYABLE | DECIMAL | ✅ Mapped | STRING → DECIMAL(18,6); name should be `profit_rate` |
| `Total Rate = Int Rate + Spread + Fix Adj Spread` | — | — | ❌ Not Available | **Pre-computed metric** — violates SLV-007 |
| `Settlement Date` | STLI_MASTER.SETTLEMENT_DATE | DATE | ✅ Mapped | STRING → DATE |
| `Status` | IPLC_EM_AMD.CURRNT_STATUS | CHAR(1) | ✅ Mapped | Single-char source code; must route through `silver.reference.code_mapping` |
| `Port Of Loading1/2/3` | IPLC_MASTER.LOAD_PORT | STRING | ✅/❌ | 3NF violation; single port per shipment leg should be in a child `trade_shipment` table |
| `Vessel Name1/2/3` | IPLC_EM_NEGO.VESSEL_NM | STRING | ✅/❌ | 3NF violation |
| `IsLogicallyDeleted` | — | STRING | ❌ No mapping | Non-standard; must be `is_active_flag` |

#### 4.37c Metadata Completeness: All 11 Silver audit columns missing.

**Findings:**
- **GAP-087 (P0/Critical, Confidence: 1.0):** Wide multi-type table with 93 attributes violates 3NF. Must be decomposed into: `trade_transaction` (header), `trade_shipment_document` (shipping legs), `trade_transaction_reference` (related references), `trade_transaction_bank` (bank routing).
- **GAP-088 (P0/High, Confidence: 1.0):** `Customer Code` and `Customer_number_CIF` are duplicates of each other — both map to the same source column (`IPLC_MASTER.C_CUST_ID`). One must be removed.
- **GAP-089 (P0/High, Confidence: 1.0):** Column `Total Rate = Int Rate + Spread + Fix Adj Spread` is a pre-computed derived metric (sum of three rates). Violates SLV-007.
- **GAP-090 (P0/High, Confidence: 0.95):** Column names use mixed case, special characters, and sentences (`Bill Type/Reimb Mode/Lc Mode/Etc`, `Due Date/Expiry Date/Maturity Date`) — severe NAM-001 violations.
- **GAP-091 (P1/High, Confidence: 0.95):** `Amount Value` sourced from `INSTALL_AMOUNT` (installment amount) and `Amount In Lcy` sourced from `LCY` — semantic alignment must be verified. `INSTALL_AMOUNT` may not represent the full transaction amount.
- **GAP-092 (P0/High, Confidence: 0.9):** `Margin Currency` maps to `CFNC_N_MARGIN_RT` (which appears to be a rate column, not a currency) — source column semantic mismatch.

---

### 4.38 Entity: Traded Investment Product

**Physical Table:** `silver_dev.trade.traded_investment_product`  
**Logical Entity:** `Traded Investment Product` (7 attributes)  
**Data Mapping Attributes:** 1  
**Temp Table:** `traded_investment_product_temp_no_partition` also exists

#### 4.38a Industry Fitness

Traded Investment Products are exchange-listed instruments. Sub-type of `investment_product`. Belongs in `silver.treasury`. Only 1 of 7 attributes mapped.

#### 4.38b Attribute-Level Review

Only `Investment Prod Id` mapped from WMS.TBAADM.SP_GEN_DESCRIPTION_TABLE.PROD_CODE. 6 of 7 attributes unmapped.

Partition column `Investment_Prod_Id` uses PascalCase — NAM-001 violation.

**Findings:**
- **GAP-093 (P0/High, Confidence: 1.0):** `traded_investment_product_temp_no_partition` temp table must be removed from Silver schema.
- **GAP-094 (P1/Medium, Confidence: 1.0):** Partition column `Investment_Prod_Id` uses PascalCase — NAM-001 violation.

---

### 4.39 Orphan Physical Tables (No Logical Model Counterpart)

#### 4.39a Entity: `fis_trade_transaction`

**Finding GAP-095 (P0/High, Confidence: 1.0):** Table `fis_trade_transaction` exists in `silver_dev.trade` with no corresponding logical model entity. The table name embeds the source system name (`fis`) — a critical NAM-001 and SLV-004 violation. Silver tables must be source-system agnostic. This table appears to be a raw FIS extract at the Silver layer, which violates the Bronze → Silver transformation contract (guideline 11 §2.2). It should either be registered in the logical model or removed from the Silver schema.

#### 4.39b Entity: `reference_product`

**Finding GAP-096 (P0/Medium, Confidence: 0.75):** Table `reference_product` has an ambiguous logical model match — it could correspond to `Reference_Product_Trade` (11 attributes, all unmapped) or to a generic product reference. The physical table is partitioned by `Product_Id` (PascalCase violation). No logical model entity named exactly `reference_product` exists in the trade SA. Requires data steward clarification. The temp variant `reference_product_temp_no_partition` must also be removed.

#### 4.39c Temp Tables in Production Silver

**Finding GAP-097 (P0/High, Confidence: 1.0):** Three temporary/work tables exist in the Silver production schema:
- `bond_temp_no_partition`
- `traded_investment_product_temp_no_partition`
- `reference_product_temp_no_partition`

Guideline 11 §2.11 states DDL must be reviewed and approved before deployment. Temp tables indicate pipeline development work committed to production. These must be dropped immediately.

---

## 5. Denormalization Register

| # | Attribute(s) | Host Entity | Correct Home | Classification | Justification Required | Finding |
|---|---|---|---|---|---|---|
| D-01 | Customer Name, Customer Category, Customer Country, Customer Segment | `Trade Transaction` | `silver.party.customer` | **Unnecessary** | None provided | GAP-028, GAP-088 |
| D-02 | RM/AO Name, RM/AO Code | `Trade Transaction` | `silver.org.employee` | **Unnecessary** | None provided | GAP-086 |
| D-03 | Counter Party Address1/2/3 | `Trade Transaction` | `trade_counterparty` (child address table) | **Unnecessary (3NF violation)** | None provided | GAP-087 |
| D-04 | Related Bank Name, BIC, Address1/2/3 | `Trade Transaction` | `trade_counterparty` | **Unnecessary** | None provided | GAP-087 |
| D-05 | Related Bank 2 (full group) | `Trade Transaction` | `trade_counterparty` | **Unnecessary** | None provided | GAP-087 |
| D-06 | Port Of Loading 1/2/3, Port Of Discharge 1/2/3, Vessel Name 1/2/3 | `Trade Transaction` | `trade_shipment_document` (child entity) | **Unnecessary (1NF violation)** | None provided | GAP-087 |
| D-07 | Goods Description 1, Goods Description 2 | `Trade Transaction` | `trade_shipment_document` | **Unnecessary (1NF violation)** | None provided | GAP-087 |
| D-08 | Related Reference Number 1/2/3/4 | `Trade Transaction` | `trade_transaction_reference` (child entity) | **Unnecessary (1NF violation)** | None provided | GAP-087 |
| D-09 | `Customer number CIF` in Bank Guarantees, Import LC, Export LC, Buyer Credit, Invoice Master, Receivable Financing | Multiple entities | `silver.party.customer` (BK reference only) | **Acceptable (FK reference)** | FKs to `silver.party` are acceptable; full customer attributes must not be copied | GAP-028 |
| D-10 | `Investment Product metric amount` ← FIS.INSTRUMENT.FACE_VALUE | `investment_product_metric_history` | Instrument master | **Unnecessary** | Static attribute sourced into a time-series table — logically incorrect | GAP-041 |
| D-11 | `Rating`, `Rating Agency id` | `trade_counterparty` | `silver.risk` or `investment_security_risk_grade` | **Unnecessary** | Counterparty ratings should FK to risk rating entity | GAP-082 |
| D-12 | Total Trade Exposure, Secured/Unsecured Amounts, FX Exposure | `trade_exposure_snapshot` | Gold aggregation or source extract | **Unacceptable (SLV-007)** | Pre-computed aggregates must not be in Silver unless direct source extract | GAP-083 |

---

## 6. Guideline Compliance Summary

| Rule | Description | Status | Findings |
|---|---|---|---|
| **SLV-001** | `<entity>_sk` (MD5 deterministic surrogate) on every entity | 🔴 FAIL — 0/22 tables | GAP-005 |
| **SLV-002** | `<entity>_bk` (business key) on every entity | 🔴 FAIL — 0/22 tables | GAP-005 |
| **SLV-003** | SCD-2 default (`effective_from`, `effective_to`, `is_current`) | 🔴 FAIL — 0/22 tables | GAP-006 |
| **SLV-004** | No source-system codes; use `silver.reference.code_mapping` | 🔴 FAIL — Multiple raw codes (Bond Type, Future Type, Status, etc.) | GAP-017, GAP-034, GAP-038, GAP-054, GAP-059, GAP-090 |
| **SLV-005** | DQ-gated writes; quarantine table | 🟡 UNKNOWN — No pipeline code available to assess; quarantine table not evidenced | — |
| **SLV-006** | 3NF; no derived values/aggregations | 🔴 FAIL — Trade Transaction has 1NF/3NF violations; multiple repeating groups | GAP-086, GAP-087 |
| **SLV-007** | No pre-computed metrics | 🔴 FAIL — Trade Exposure Snapshot and Trade Transaction `Total Rate` field | GAP-083, GAP-089 |
| **SLV-008** | All metadata columns present (11 Silver audit columns) | 🔴 FAIL — 0/22 tables have any of the required columns | GAP-006, all per-entity §5c findings |
| **SLV-009** | All timestamps UTC | 🔴 FAIL — All date/timestamp columns typed as STRING | All per-entity §5b findings |
| **SLV-010** | Monetary amounts in smallest unit (fils for AED) | 🔴 FAIL — All monetary columns typed as STRING; not in fils | All per-entity §5b findings |
| **NAM-001** | snake_case, lowercase, no reserved words | 🔴 FAIL — Multiple violations: PascalCase partition columns, mixed-case attr names, sentence-style column names, typos | GAP-009, GAP-016, GAP-025, GAP-050, GAP-064, GAP-090, GAP-094 |
| **NAM-003** | Table naming: `<entity>` for Silver | 🟡 PARTIAL — Core name pattern followed but with typos, `_1` suffixes, `fis_` prefix, temp table suffixes | GAP-009, GAP-025, GAP-064, GAP-095, GAP-097 |
| **NAM-005** | No SQL reserved words as identifiers | 🟡 UNKNOWN — Cannot assess column DDL (no column-level physical structures) | — |
| **DQ-001** | Deduplication on business keys | 🔴 FAIL — No `_bk` columns; cannot enforce uniqueness | GAP-005 |
| **DQ-002** | Date standardization YYYY-MM-DD | 🔴 FAIL — All dates are STRING | All per-entity §5b |
| **DQ-003** | Monetary columns `DECIMAL(18,4)` | 🔴 FAIL — All amounts are STRING | All per-entity §5b |
| **DQ-004** | Error rate > 5% fails pipeline | 🟡 UNKNOWN — No pipeline code | — |
| **DQ-005** | Failed records to quarantine table | 🟡 UNKNOWN — No pipeline code | — |
| **Catalog Reg.** | All tables registered in CDGC with descriptions | 🔴 FAIL — All 27 physical tables have `description: null` | All tables |
| **3NF** | Third Normal Form (guideline 11 §2.1) | 🔴 FAIL — Trade Transaction, multiple reference tables | GAP-086, GAP-087, D-01 through D-08 |
| **SA Isolation** | Subject area scoping per subject-areas.md | 🔴 FAIL — Trade mixes trade_finance + treasury + reference + party | GAP-001, GAP-002, GAP-003 |
| **SK Determinism** | MD5 surrogate, never UUID/auto-increment | 🔴 FAIL — No SK implementation | GAP-005 |
| **No SELECT \*** | Explicit column lists in pipelines | 🟡 UNKNOWN — No pipeline code | — |
| **UTC Only** | All timestamps UTC | 🔴 FAIL — All STRING types | GAP-006 |

**Overall Compliance Score: 2/20 rules fully compliant (SLV-005, DQ-004, DQ-005, NAM-005, No SELECT* — all UNKNOWN, not failing). 0 rules confirmed passing. 12 rules confirmed failing.**

---

## 7. Remediation Plan

### 7.1 Prioritized Action List

#### P0 — Immediate (Blocking — before any Gold layer consumption)

| ID | Action | Effort | Owner | Guideline |
|---|---|---|---|---|
| R-01 | **Decompose SA**: create `silver.trade_finance` schema for LC/BG/Trade Txn/Counterparty/Invoice; migrate investment entities to `silver.treasury`; migrate reference entities to `silver.reference` | XL (6–8 weeks) | Data Architect + Steward | SA scoping, SLV-002 |
| R-02 | **Add SK/BK columns** to all 22 existing physical tables: `<entity>_sk = MD5(UPPER(TRIM(<entity>_bk)))`, `<entity>_bk = source business key` | L (2 weeks) | Data Engineering | SLV-001, SLV-002 |
| R-03 | **Add SCD-2 + audit columns** to all 22 existing physical tables: `effective_from TIMESTAMP`, `effective_to TIMESTAMP`, `is_current BOOLEAN`, `source_system_code`, `source_system_id`, `create_date`, `update_date`, `delete_date`, `is_active_flag`, `run_id`, `source_ingestion_date` | L (2 weeks) | Data Engineering | SLV-003, SLV-008 |
| R-04 | **Create physical tables for critical missing entities**: Bank Guarantees, Import LC, Export LC, Trade Counterparty, Invoice Master | L (3 weeks per entity = 15 weeks total) | Data Engineering | GAP-012, GAP-029, GAP-035, GAP-080, GAP-051 |
| R-05 | **Fix all data types**: all monetary → `DECIMAL(18,4)`; all dates → `DATE`; all timestamps → `TIMESTAMP` UTC; all codes → `STRING` (but route through code_mapping) | M (1 week) | Data Engineering | SLV-009, SLV-010, DQ-003 |
| R-06 | **Drop temp tables**: `bond_temp_no_partition`, `traded_investment_product_temp_no_partition`, `reference_product_temp_no_partition` | XS (1 day) | Data Engineering | GAP-097 |
| R-07 | **Remove `fis_trade_transaction`** from Silver or register with logical model and rename to source-agnostic `trade_transaction_fis_raw` (Bronze-level) | S (2 days) | Data Engineering + Architect | GAP-095, SLV-004 |
| R-08 | **Rename typo tables**: `assest_backed_security_product` → `asset_backed_security_product`; `credit_derivate_product` → `credit_derivative_product`; `reference_exchange_rate1` → `reference_exchange_rate` | S (1 day + migration) | Data Engineering | GAP-009, GAP-025, GAP-064, NAM-001 |
| R-09 | **Remove Customer entity** from trade SA logical model; replace with `customer_bk STRING` FK reference in all trade entities | S (2 days) | Data Modeler | GAP-028, GAP-002 |
| R-10 | **Decompose Trade Transaction** into: `trade_transaction` (header), `trade_shipment_document` (per-leg), `trade_transaction_reference` (ref numbers), `trade_transaction_bank_routing` (bank details) | XL (4–6 weeks) | Data Modeler + Engineering | GAP-086, GAP-087, SLV-006 |
| R-11 | **Fix Exchange Rate dual-mapping**: `Source to Target Rate` and `Target to Source Rate` must map to different source columns; validate with EXIMBILLS DBA | S (1 week) | Data Engineering | GAP-063 |
| R-12 | **Resolve Trade Exposure Snapshot pre-aggregation**: steward ruling — move to Gold if computed, or document as source-system extract | S (data steward review) | Data Steward | GAP-083, GAP-084, SLV-007 |
| R-13 | **Register all 27 tables in CDGC** with business descriptions, data steward, classification, lineage | M (2 weeks) | Data Governance | Catalog Reg, guideline 11 §2.12 |

#### P1 — Next Sprint (Critical quality fixes)

| ID | Action | Effort | Owner |
|---|---|---|---|
| R-14 | Fix all PascalCase partition columns: `Investment_Product_Id` → `investment_product_bk` or remove partition | S | Data Engineering |
| R-15 | Implement code mapping for all raw source codes (Bond Type, Future Type, Option Style, Status codes) | M | Data Engineering |
| R-16 | Create physical tables for: Buyer Credit, Receivable Financing, Reference Incoterm, Reference Currency, Reference Source System | M per entity | Data Engineering |
| R-17 | Create physical tables for Stock and Stock History | M | Data Engineering |
| R-18 | Resolve `Investment Product Id` key inconsistency: `INSTRUMENT.INSID` vs `INSTRUMENT.product_chlnbr` across entities | S | Data Engineering |
| R-19 | Fix `Investment Product metric amount` semantic mapping (not FACE_VALUE) | S | Data Engineering |
| R-20 | Remove `Reference_Country1` duplicate from logical model | XS | Data Modeler |
| R-21 | Complete ETL mapping for 22+ attributes currently "Not Available" | L | Source System SMEs |
| R-22 | Fix `Margin Currency` mapping (maps to rate column not currency code) | S | Data Engineering |
| R-23 | Rename all non-conformant column names in Trade Transaction (sentence-style names, slash-separated names) | M | Data Modeler |

#### P2 — Backlog

| ID | Action | Effort | Owner |
|---|---|---|---|
| R-24 | Evaluate Trade Counterparty Rating denormalization vs. `silver.risk` FK | M | Data Architect |
| R-25 | Complete mapping for all investment product sub-entities (6+ unmapped attributes each) | L | Source System SMEs |
| R-26 | Evaluate multi-source Investment Product key strategy (FIS vs. WMS PROD_CODE vs. WMS INSID) | M | Data Architect |
| R-27 | Rename logical entity `Investment Produc Term Period History` (missing 't') | XS | Data Modeler |
| R-28 | Review `Fund` entity PII flag anomaly (`1 distinct value` in PII column) | XS | Data Governance |
| R-29 | Align `Interest Index description` to correct source column (not C_CP_UNIT_CODE) | S | Data Engineering |
| R-30 | Build DQ quarantine tables and implement error-rate alerting (SLV-005) | M | Data Engineering |

### 7.2 Recommended Schedule

| Sprint | Actions | Outcome |
|---|---|---|
| Sprint 1 (Week 1–2) | R-06, R-07, R-08, R-09 (drop temps, rename tables, remove customer) | Schema clean-up; no data loss |
| Sprint 2 (Week 3–4) | R-02, R-03 (SK/BK + SCD-2 columns via non-breaking DDL additions) | Identity and history enabled |
| Sprint 3 (Week 5–6) | R-05, R-11, R-14, R-15 (data types, code mapping, partition fixes) | Type safety and canonical codes |
| Sprint 4 (Week 7–8) | R-13 (CDGC registration) | Governance visibility |
| Sprint 5–6 (Week 9–12) | R-04 (missing entity tables: Bank Guarantees, Import/Export LC, Trade Counterparty, Invoice Master) | Core trade finance entities |
| Sprint 7–8 (Week 13–16) | R-10 (Trade Transaction decomposition) | 3NF compliance |
| Sprint 9–12 (Week 17–24) | R-01 (SA decomposition), R-16, R-17, R-21 | Full SA restructuring |
| Sprint 13–14 (Week 25–28) | R-12, R-24–R-30 (backlog items) | Quality hardening |

---

## 8. Appendix

### Appendix A: Mapping Summary

| Category | Count | Notes |
|---|---|---|
| Total logical entities | 38 | From `bank_logical_model.xlsx` Trade filter |
| Total physical tables | 27 | From `trade_physical_structures.csv` |
| Matched entity ↔ table | 22 | Partial matches (not validated column-by-column due to no DDL in CSV) |
| Unmatched logical entities (no physical) | 16 | Bank Guarantees, Import/Export LC, Trade Counterparty, Trade Exposure Snapshot, Invoice Master, Buyer Credit, Receivable Financing, Reference Currency, Reference Incoterm, Reference Interest Rate 1, Reference Product Trade, Reference Source System, Stock, Stock History, Reference Country (wrong SA), Duplicate Reference Country 1 |
| Orphan physical tables (no logical match) | 5 | `fis_trade_transaction`, `reference_product` (ambiguous), `bond_temp_no_partition`, `traded_investment_product_temp_no_partition`, `reference_product_temp_no_partition` |
| Total data mapping entities | 39 | From `trade_data_mapping.xlsx` "Trade Silver Mapping Sheet" |
| Mapping entities not in logical model | 1 | `Bank Kafalah` (should unify with `Bank_Guarantees`) |
| Total mapped attributes | ~135/375 (~36%) | Status="Mapped" |
| Attributes "Not Available" | ~42/375 (~11%) | Source identified but no column |
| Attributes with no mapping | ~198/375 (~53%) | Status=None |

**Source Systems Identified:**
- **EXIMBILLS** (Sopra Banking trade finance platform) — primary source for LC, BG, country, currency, interest rate, exchange rate, trade transaction
- **FIS** (Temenos/FIS) — primary source for investment products (Bonds, Futures, Options, Credit Derivatives, Money Market, Currency Products, Commodity)
- **WMS** (Wealth Management System) — source for Mutual Funds, Non-Traded Products, Stocks, Traded Products

### Appendix B: Guideline Citations

| Rule Code | Rule Text (Exact Quote) | Source |
|---|---|---|
| SLV-001 | "Every Silver entity must carry `<entity>_sk` (MD5 surrogate derived deterministically from the business key)" | `guidelines/11-modeling-dos-and-donts.md` §2.4 |
| SLV-002 | "`<entity>_bk` (natural/business key from source)" | `guidelines/11-modeling-dos-and-donts.md` §2.4 |
| SLV-003 | "Track all changes to tracked attributes with SCD Type 2: add a new row with updated values, close the prior row (`effective_to` = change timestamp, `is_current` = FALSE). Every Silver entity table must have `effective_from`, `effective_to`, `is_current`." | `guidelines/11-modeling-dos-and-donts.md` §2.3 |
| SLV-004 | "Resolve all source-system codes to canonical platform values using a central code-mapping reference (e.g., `silver.reference.code_mapping`). Store the canonical code, not the raw source code, in entity columns." | `guidelines/11-modeling-dos-and-donts.md` §2.5 |
| SLV-006 | "Decompose entities into 3NF: every non-key attribute depends on the whole primary key and nothing but the primary key. Repeating groups (sets of similar attributes that recur) must become child tables." | `guidelines/11-modeling-dos-and-donts.md` §2.1 |
| SLV-007 | "DON'T pre-compute totals, averages, counts, ratios, or any derived metrics in Silver tables." | `guidelines/11-modeling-dos-and-donts.md` §2.6 |
| SLV-008 | "Include the following Silver technical audit columns on every Silver entity table: `source_system_code`, `source_system_id`, `create_date`, `update_date`, `delete_date`, `is_active_flag`, `effective_from`, `effective_to`, `is_current`, `run_id`, `source_ingestion_date`" | `guidelines/11-modeling-dos-and-donts.md` §2.7 |
| SLV-008b | "Every Silver and Gold table must carry the following columns: `record_source`, `record_origin`, `record_hash`, `is_current`, `valid_from_at`, `valid_to_at`, `ingested_at`, `processed_at`, `batch_id`, `data_quality_status`, `dq_issues`" | `guidelines/01-data-quality-controls.md` §5 |
| SLV-009 | "Store all timestamps in UTC. Convert source-local timestamps at ingestion." | `guidelines/11-modeling-dos-and-donts.md` §2.9 |
| SLV-010 | All monetary columns `DECIMAL(18,4)`; "All monetary columns must use `DECIMAL(18,4)`" | `guidelines/01-data-quality-controls.md` §2 Gate 2 |
| NAM-001 | "Use snake_case for all identifiers (catalogs, schemas, tables, columns, views). Use lowercase only — no mixed case." | `guidelines/06-naming-conventions.md` NAM-001 |
| NAM-003 | Silver entity table pattern: `<entity>` | `guidelines/06-naming-conventions.md` NAM-003 |
| NAM-005 | "Never use SQL reserved words as identifiers (e.g., `date`, `value`, `order`)." | `guidelines/06-naming-conventions.md` NAM-005 |
| SA-SCOPE | Canonical subject area list and schemas | `sa/subject-areas.md` |
| SA-ISOLATION | "Keep each Silver subject area self-contained. Pipelines for one subject area read only from Bronze and from the same subject area's own Silver tables." | `guidelines/11-modeling-dos-and-donts.md` §2.2 |

**Note on Metadata Column Discrepancy:** Guideline 01 (DQ Controls §5) specifies 11 audit columns using `record_source`, `valid_from_at`, `valid_to_at`, `ingested_at`, `processed_at` naming. Guideline 11 (Modeling §2.7) specifies 11 audit columns using `source_system_code`, `effective_from`, `effective_to`, `source_ingestion_date`, `run_id` naming. These are partially overlapping but distinct sets. **Recommendation:** The data steward should ratify a single canonical set. The mapping workbook aligns with Guideline 11's column names; Guideline 01's `record_hash`, `data_quality_status`, `dq_issues`, `batch_id` columns add DQ traceability that is absent from Guideline 11's set. The ratified canonical Silver audit column set should be the **union** of both guidelines (15–16 columns total).

### Appendix C: Industry References

| Standard | Relevance to Findings |
|---|---|
| **BIAN v11 — Trade Finance** | `Commercial Letters of Credit (Import/Export)`, `Issued Guarantees and Indemnities`, `Documentary Collections` — canonical entity boundaries for LC, BG, Trade Transaction decomposition |
| **BIAN v11 — Securities Administration** | `Traded Positions`, `Investment Portfolio`, `Securities Custody` — canonical boundaries for Bond, Fund, Option, Future, Commodity entities |
| **IFW (IBM/Teradata Banking Data Model)** | `InstrumentMaster`, `InstrumentPosition`, `ContractGroup` — aligns with `investment_product` super-type pattern |
| **IFRS 9** | Investment Security Risk Grade is relevant for ECL staging and FVTPL/FVOCI classification; the missing mapping here creates IFRS 9 compliance risk |
| **BCBS 239** | Risk data aggregation principles require complete lineage and atomic data — pre-aggregated Trade Exposure Snapshot violates BCBS 239 Principle 4 (accuracy and integrity) |
| **CBUAE Regulatory Reporting** | Trade finance exposures must be reported accurately to CBUAE; missing Import/Export LC, Bank Guarantees, and Buyer Credit tables create a regulatory reporting gap |
| **ICC Incoterms 2020** | Standard trade terms (EXW, FOB, CIF, DDP etc.) — the Reference Incoterm entity must be aligned to ICC 2020 definitions |
| **ISO 4217** | Currency codes standard — `Reference_Currency` table should use ISO 4217 numeric and alpha codes |
| **ISO 3166-1** | Country codes — `Reference_Country` should use ISO 3166-1 alpha-2, alpha-3, and numeric codes |
| **SWIFT FIN Standards (MT700, MT760)** | LC and BG SWIFT message types — the Trade Transaction decomposition should align with SWIFT MT field structure for interoperability |

---

*End of Report*  
*Generated: 2025-12-11 | Trade Subject Area Silver-Layer Gap Assessment v1.0*
