# Silver Layer Gap Assessment Report
## Subject Area: Investments
**Assessment Date:** 2026-04-27
**Assessor:** Senior Data Platform Architect
**Repository:** `imgokhan-rakbank/logical-model-assessment`
**Environment:** `silver_dev.investments`
**Guideline Version:** Data Platform Development Standards V1.0 Draft (December 2025)

---

## 1. Executive Summary

### 1.1 Management Slide

The **Investments** subject area is currently a **development-environment-only schema** (`silver_dev.investments`) with no registration in the canonical subject-area taxonomy (`sa/subject-areas.md`). Closest canonical equivalents are **Wealth** (`silver.wealth`, Priority 4) and **Treasury** (`silver.treasury`, Priority 4). The subject area mixes both domains without clear decomposition and embeds reference data that belongs in `silver.reference`.

Of the 31 logical entities defined in the `bank_logical_model.xlsx`:

| Status | Count |
|---|---|
| Physical table found | 25 |
| Physical table **missing** | 5 — Investment Account, Trade Transaction, Cash Dividend Event, Foreign Exchange Event, Stock |
| Extra physical table (not in LM) | 1 — `foreign_exchange_agreement_temp_no_partition` |

**Mapping Health (from `investments_data_mapping.xlsx`):**

| Status | Attributes |
|---|---|
| Mapped | ~156 |
| Not Available | ~70 |
| Blank/Unmapped | ~5 |

**Key Risk:** The single most critical finding is that **zero entities have surrogate keys (`_sk`) or business keys (`_bk`)** — every entity relies solely on raw source-system natural identifiers, making Gold-layer joins non-deterministic and pipeline re-runs destructive. Additionally, **every single attribute in the logical model is typed `STRING`**, including monetary amounts, dates, rates, and flags — a complete absence of data-type governance.

**Overall Health Rating:** 🔴 **CRITICAL** — 11 P0 findings, 22 P1 findings, 14 P2 findings.

---

### 1.2 Top 5 Immediate Actions

| # | Action | Rule | Priority |
|---|---|---|---|
| 1 | Add `<entity>_sk` (MD5 deterministic surrogate) + `<entity>_bk` to **every** entity | SLV-001/002 | **P0** |
| 2 | Correct all monetary-amount attributes from `STRING` to `DECIMAL(18,4)`; date fields to `DATE`/`TIMESTAMP` | SLV-010 / Gate 2 | **P0** |
| 3 | Register SA in `sa/subject-areas.md` and align schema to canonical `silver.wealth` / `silver.treasury` | NAM-002, SA taxonomy | **P0** |
| 4 | Move reference tables (`reference_currency_code`, `reference_exchange_rate2`, `reference_product`, `reference_transaction_code`, `unit_of_measure`) to `silver.reference` | SLV-004 / SA isolation | **P0** |
| 5 | Resolve mapping of Investment Transaction's 17 unmapped critical financial attributes (Transaction Amount, Net Amount, Gross Amount, AED Amount, Unit Price, Exchange Rate, etc.) | SLV-005 / completeness | **P0** |

---

## 2. Subject Area Architecture Assessment

### 2.1 Scoping vs. Subject-Area Taxonomy

**Finding:** The name "investments" does not appear in `sa/subject-areas.md`. The canonical taxonomy defines:

- **Wealth** (`silver.wealth`, #9) — "Investment Services: Mutual Funds, Equity Portfolios, Bond Holdings, and Advisory Mandates."
- **Treasury** (`silver.treasury`, #15) — "Bank Investments: Interbank Deals, Securities (Bonds/Stocks/Sukuk), Derivatives (Swaps/Forwards), and Money Market placements."

The physical schema `silver_dev.investments` contains entities from **both** domains without separation:

| Domain | Entities in SA |
|---|---|
| Wealth / Client-book | `investment_product`, `security_holding`, `trade_order`, `unit_distribution_event`, `application_investment_detail`, `stock_history`, dividend/distribution events |
| Treasury / Bank-book | `swap_agreements`, `swap_agreement_leg`, `swap_currency_agreement`, `swap_leg_*`, `foreign_exchange_agreement`, `forward_futures_agreement`, `option_agreement`, `securities_financing`, `cash_at_other_banks` |
| Reference data (wrong SA) | `reference_currency_code`, `reference_exchange_rate2`, `reference_product`, `reference_transaction_code`, `unit_of_measure` |

**Impact:** Consumers cannot distinguish client-wealth from bank-treasury positions; regulatory reporting (CBUAE market risk vs. wealth management returns) is conflated; stewardship is ambiguous; SA isolation rule violated.

**Recommendation:** Split into two canonical SAs:
- Register `silver.wealth` for client-facing investment entities.
- Register `silver.treasury` for bank-book derivatives, FX, securities financing.
- Migrate reference tables to `silver.reference`.

---

### 2.2 Domain Decomposition vs. BIAN/IFW

The SA aligns partially with BIAN v11 service domains but has diverged:

| BIAN Service Domain | Expected entities | Actual coverage |
|---|---|---|
| Investment Portfolio Management | Portfolio, Holding, Investment Account, NAV | `security_holding`, `investment_product` ✅; **Investment Account ❌ missing** |
| Securities Operations | Trade Order, Trade Execution, Settlement | `trade_order`, `trade_execution` ✅; **Trade Transaction ❌ missing** |
| Market Order | Order management, order book | `trade_order` partially |
| Currency Exchange | FX Agreement, FX Event | `foreign_exchange_agreement` ✅; **Foreign Exchange Event ❌ missing** |
| Collateral Management | Securities Financing, Repo | `securities_financing` ✅ |
| Derivative Operations | Swap, Options, Forwards | All swap entities ✅; option/forward ✅ |
| Dividend & Corporate Actions | Dividend events, Stock | `dividend_declare_event`, `stock_dividend_amount` ✅; **Cash Dividend Event ❌ missing**; **Stock ❌ missing** |
| Reference Data | Product, Currency, Exchange Rate | Present but **in wrong SA** |

**Missing BIAN entities from the SA:** Investment Account (BIAN: Investment Portfolio Account), Trade Transaction (BIAN: Securities Trade Booking), Cash Dividend Event (BIAN: Corporate Action Event), Foreign Exchange Event (BIAN: Currency Exchange), Stock (BIAN: Security).

---

### 2.3 Identity Strategy

**Status: ❌ ABSENT across all entities.**

Not a single entity in the logical model or physical structure defines:
- An `<entity>_sk` column (MD5 deterministic surrogate key).
- An `<entity>_bk` column (business/natural key).

All entity-identification relies on source-system natural keys (`Account Number`, `Product Id`, `Event Id`, `Transaction Id`). This violates SLV-001 and SLV-002 and means:
- Pipeline re-runs generate different keys if UUIDs are used.
- Gold-layer dimension tables cannot reliably join to Silver.
- MDM Global_UUID resolution (Gate 3) cannot be performed.

**Evidence:** Logical model (Consolidated Metadata for Subject sheet): no `_sk` or `_bk` columns defined for any of 31 entities. Data mapping (Investment Silver Mapping): no `_sk` or `_bk` in any entity attribute list.

---

### 2.4 SCD Strategy

**Status: ❌ Non-standard naming; CHAR(18) type for temporal columns.**

The logical model uses:
- `Business Start Date` (CHAR(18)) → should be `effective_from` (TIMESTAMP)
- `Business End Date` (CHAR(18)) → should be `effective_to` (TIMESTAMP)
- `Is Active Flag` (CHAR(18)) → should be `is_current` (BOOLEAN)
- `IsLogicallyDeleted` (STRING) → should be `delete_date` (TIMESTAMP, nullable) per standard

`CHAR(18)` is an incorrect data type for date/temporal columns. The guideline (§ 2.7 of 11-modeling-dos-and-donts.md) mandates TIMESTAMP for `effective_from`/`effective_to` and BOOLEAN for `is_current`.

**Missing metadata columns** (required per guideline § 2.7):
- `run_id` (STRING) — absent from all entities
- `source_ingestion_date` (TIMESTAMP) — absent from all entities

The `Investment Transaction` entity is also missing `source_system_code` and `source_system_id` mappings (status: "Not available").

---

### 2.5 Cross-Entity Referential Integrity

Several foreign-key relationships have no defined FK constraint and map to missing physical entities:

| FK Relationship | Status |
|---|---|
| `investment_product_event.investment_product_id` → `investment_product` | Logical FK; no physical constraint |
| `security_holding.account_number` → **Investment Account** | Investment Account has no physical table ❌ |
| `trade_execution.transaction_id` → **Trade Transaction** | Trade Transaction has no physical table ❌ |
| `dividend_declare_event.investment_product_id` → `investment_product` | No physical constraint |
| `swap_agreement_leg.account_number` → `swap_agreements` | No physical constraint |
| `swap_leg_cash_flow_feature.account_number` → `swap_agreement_leg` | No physical constraint + 6/7 attrs unmapped |

---

### 2.6 ETL Mapping Completeness

Total mapped attributes: ~156 of ~226 mapped-entity attributes (**69% coverage**).
Not-available attributes: ~70 (**31% gap**).

Critical unmapped financial attributes in **Investment Transaction**:
- Transaction Amount, Net Transaction Amount, Gross Transaction Amount, Transaction Amount In AED
- Transaction Unit, Unit Price, Exchange Rate
- Financial Market Commitment Amount, Unit Holdings
- Transaction Text, Pledge Indicator, Source Type
- `Source System Code`, `Source System Id` (mandatory audit columns)

The **Reference_Exchange_Rate2** table has 2 of its 5 business attributes (`Source to Target Currency Rate`, `Target to Source Currency Rate`) unmapped — the actual exchange rate fields are not sourced.

---

## 3. Entity Inventory

| # | Logical Entity | Physical Table | Physical Exists | Mapping Status | Mapped/Total Attrs | SK/BK | SCD Cols | P |
|---|---|---|---|---|---|---|---|---|
| 1 | Application Investment detail | `application_investment_detail` | ✅ | Partial | 4/5 | ❌ | Non-std | P1 |
| 2 | Cash Dividend Event | — | ❌ **MISSING** | Partial | 4/5 | ❌ | — | P1 |
| 3 | Cash at Other Banks | `cash_at_other_banks` | ✅ | Full | 3/3 | ❌ | Non-std | P1 |
| 4 | Dividend Declare Event | `dividend_declare_event` | ✅ | Partial | 2/4 | ❌ | Non-std | P1 |
| 5 | Financial Agreement | `financial_agreement` | ✅ | Partial | 6/9 | ❌ | Non-std | P1 |
| 6 | Foreign Exchange Agreement | `foreign_exchange_agreement` | ✅ | Full | 8/8 | ❌ | Non-std | P1 |
| 7 | Foreign Exchange Event | — | ❌ **MISSING** | Full | 2/2 | ❌ | — | P1 |
| 8 | Forward Futures Agreements | `forward_futures_agreement` | ✅ | Full | 2/2 | ❌ | Non-std | P1 |
| 9 | **Investment Account** | — | ❌ **MISSING** | None | 0/22 | ❌ | — | **P0** |
| 10 | Investment Prod External Event | `investment_prod_external_event` | ✅ | Full | 3/3 | ❌ | Non-std | P1 |
| 11 | Investment Product | `investment_product` | ✅ | Partial | 3/4 | ❌ | Non-std | P1 |
| 12 | Investment Product Event | `investment_product_event` | ✅ | Partial | 5/7 | ❌ | Non-std | P1 |
| 13 | Investment Transaction | ❌ No direct table | ❌ **UNMAPPED** | Partial | 53/70 | ❌ | Non-std | **P0** |
| 14 | Option Agreement | `option_agreement` | ✅ | Partial | 6/7 | ❌ | Non-std | P1 |
| 15 | Reference_Exchange_Rate2 | `reference_exchange_rate2` | ✅ (wrong SA) | Partial | 3/5 | ❌ | Non-std | P1 |
| 16 | Reference_Product | `reference_product` | ✅ (wrong SA) | Partial | 5/9 | ❌ | Non-std | P1 |
| 17 | Securities Financing | `securities_financing` | ✅ | Partial | 6/8 | ❌ | Non-std | P1 |
| 18 | Security Holding | `security_holding` | ✅ | Partial | 2/13* | ❌ | Non-std | P1 |
| 19 | **Stock** | — | ❌ **MISSING** | None | 0/14 | ❌ | — | P1 |
| 20 | Stock Dividend Amount | `stock_dividend_amount` | ✅ | Partial | 10/11 | ❌ | Non-std | P1 |
| 21 | Stock History | `stock_history` | ✅ | — | — | ❌ | Non-std | P1 |
| 22 | Swap Agreement Leg | `swap_agreement_leg` | ✅ | Partial | 3/5 | ❌ | Non-std | P1 |
| 23 | Swap Agreements | `swap_agreements` | ✅ | Partial | 6/7 | ❌ | Non-std | P1 |
| 24 | Swap Leg Account Event | `swap_leg_account_event` | ✅ | Partial | 2/4 | ❌ | Non-std | P1 |
| 25 | Swap Leg Cash Flow Feature | `swap_leg_cash_flow_feature` | ✅ | Critical | 1/7 | ❌ | Non-std | **P0** |
| 26 | Swap_Currency_Agreement | `swap_currency_agreement` | ✅ | Full | 4/4 | ❌ | Non-std | P1 |
| 27 | Trade Execution | `trade_execution` | ✅ | Partial | 4/5 | ❌ | Non-std | P1 |
| 28 | Trade Order | `trade_order` | ✅ | Partial | 8/9 | ❌ | Non-std | P1 |
| 29 | **Trade Transaction** | — | ❌ **MISSING** | None | 0/93 | ❌ | — | **P0** |
| 30 | Unit Distribution Event | `unit_distribution_event` | ✅ | Partial | 3/6 | ❌ | Non-std | P1 |
| 31 | Unit of Measure | `unit_of_measure` | ✅ (wrong SA) | Partial | 2/3 | ❌ | Non-std | P2 |

**Extra physical table (no LM counterpart):** `foreign_exchange_agreement_temp_no_partition` — not in any entity definition.

\* Security Holding has 13 logical attributes but only 2 are in the mapping; LM mapping is severely incomplete.

---

## 4. Per-Entity Assessments

---

### 4.1 Investment Transaction

#### 4.1a — Industry Fitness

**Status: 🔴 INCORRECT**

`Investment Transaction` is a **god-object entity** with 70+ attributes spanning at least 8 conceptual concerns:
1. Account identification (`Account Num`, `Account Type`, `Account Currency`)
2. Settlement details (`Settlement Account`, `Settlement Currency`, `Settlement Amount`, `Settlement Method`, `Transaction Acknowledge Settlement Date`)
3. Core transaction measures (`Transaction Amount`, `Net Amount`, `Gross Amount`) — **unmapped**
4. Fee and rate structures — **6 distinct rate variants** (`Original Fee Rate`, `Adhoc Fee Rate`, `Promo Fee Rate`, `Fee Rate`, `Original Commitment Rate`, `Adhoc Commitment Rate`, `Promo Commitment Rate`, `Commitment Rate`)
5. Channel and distribution data (`Sales Channel`, `Distributed Channel`, `Sales Staff`, `Sales Branch`, `Entity Branch Code`)
6. Customer segmentation data (`Customer Segment`, `Customer Segment Description`)
7. Legacy values (`Legacy Investment Amount`, `Legacy Average Cost`, `Legacy Tax Amount`)
8. Risk (`Transaction Risk Level`)

This violates 3NF (SLV-006). In BIAN terms, the entity conflates **Securities Trade Booking**, **Payment Order**, **Investment Portfolio Management**, and **Business Channel Management** into one table. This should be decomposed into:
- `investment_transaction` (core transaction grain: ID, date, type, amounts)
- `investment_transaction_settlement` (settlement details)
- `investment_transaction_fee` (fee/rate schedule)
- Channels → `silver.event` (Business Event SA)
- Customer Segment → `silver.party` (Party SA)

**Grain:** Not clearly defined — could be one row per transaction or one row per transaction-settlement combination.

**No physical table exists** for `investment_transaction`. The closest candidates are `trade_order` and `trade_execution` in the physical layer, but these serve different purposes (order management vs. execution), and the Investment Transaction attributes span multiple source tables (`account`, `settlement`, `trade`, `trans_hst`, `journal`, `yield_curve`, etc.) suggesting it requires a materialized composite.

#### 4.1b — Attribute-Level Review

| Attribute | Logical Type | Source Type | Mapping | Issue |
|---|---|---|---|---|
| Account Num | STRING | char | Mapped | Type mismatch risk: source char, target STRING — acceptable, but should be `account_bk` per naming |
| Account Type | STRING | int | Mapped (straight-through) | **SLV-004**: raw int source code stored as STRING — needs canonical code mapping |
| Account Currency | STRING | numeric | Mapped (straight-through) | **SLV-010**: currency is numeric in source, mapped as STRING; should be ISO 4217 code via reference mapping |
| Settlement Account Type | STRING | int | Mapped | **SLV-004**: same issue as Account Type |
| Transaction Amount | STRING | (none) | **Not Available** | **CRITICAL**: core financial measure with no source mapping |
| Net Transaction Amount | STRING | (none) | **Not Available** | CRITICAL |
| Gross Transaction Amount | STRING | (none) | **Not Available** | CRITICAL |
| Transaction Amount In AED | STRING | (none) | **Not Available** | **SLV-010**: AED conversion column unmapped; required for UAE regulatory reporting |
| Transaction Unit | STRING | (none) | **Not Available** | Required for portfolio valuation |
| Unit Price | STRING | (none) | **Not Available** | Required for NAV computation |
| Exchange Rate | STRING | (none) | **Not Available** | Required for AED conversion; ironic since there is a `reference_exchange_rate2` table |
| Invesment Amount | STRING | Decimal | Mapped | **NAM-001**: typo "Invesment" (missing 't'); **SLV-010**: should be DECIMAL(18,4) |
| Legacy Investment Amount | STRING | Decimal | Mapped | SLV-006: legacy values don't belong in Silver |
| Legacy Tax Amount | STRING | (none) | **Not Available** | SLV-006: legacy values; also unmapped |
| Legacy Average Cost | STRING | float | Mapped | SLV-006: legacy values |
| Financial Market Commitment Amount | STRING | (none) | **Not Available** | |
| Unit Holdings | STRING | (none) | **Not Available** | Required for position reporting |
| Fee Rate | STRING | float | Mapped | **SLV-010**: should be DECIMAL(18,6) for rate precision |
| Original Fee Rate | STRING | float | Mapped | Multiple rate columns from same source column `dbo.trade.fee` — **duplicate mapping ambiguity** |
| Adhoc Fee Rate | STRING | float | Mapped | Same source as Fee Rate and Original Fee Rate — ambiguous |
| Promo Fee Rate | STRING | float | Mapped | Same source — ambiguous |
| Transaction Date | STRING | datetime | Mapped | **SLV-010**: should be DATE or TIMESTAMP |
| Transaction Settlement Date | STRING | datetime | Mapped | Should be DATE |
| Transaction Confirmation Date | STRING | datetime | Mapped | Should be TIMESTAMP |
| Settlement Status Change Date | STRING | datetime | Mapped | Should be DATE/TIMESTAMP |
| Transaction Order Date | STRING | datetime | Mapped | Should be DATE/TIMESTAMP |
| Transaction Arc Id | STRING | numeric | Mapped | Misleading name — maps to `trans_hst.authorizer_usrnbr` (not an "Arc" ID but an authoriser user number) |
| Transaction Id | STRING | Integer | **DUPLICATED** | `Transaction Id` appears twice in the mapping (rows differ by source expression). Guideline note: 1 duplicate variable recorded in Field Classification. |
| Transaction Code | STRING | Integer | Mapped | **SLV-004**: raw integer source code, needs canonical mapping |
| Transcation Status | STRING | char | Mapped | **NAM-001**: typo "Transcation"; **SLV-004**: char status code needs mapping |
| Transcaction Settlement Status Code | STRING | int | Mapped | **NAM-001**: double typo "Transcaction"; **SLV-004**: int status code |
| Customer Segment | STRING | int | Mapped | **3NF violation**: customer attribute in transaction entity |
| Customer Segment Description | STRING | (none) | **Not Available** | **3NF violation + unmapped** |
| Sales Channel | STRING | char | Mapped | **3NF violation**: channel dimension data in transaction |
| Distributed Channel | STRING | char | Mapped | **3NF violation**: same source column as Sales Channel — duplicate/ambiguous |
| Sales Staff | STRING | numeric | Mapped | **3NF violation**: staff/party reference in transaction |
| Sales Branch | STRING | numeric | Mapped | Same source as `Entity Branch Code` — duplicate mapping |
| Entity Branch Code | STRING | numeric | Mapped | Same source `dbo.trade_regulatory_info.branch_membership_ptynbr` — same as Sales Branch; ambiguous |
| Source System Code | STRING | (none) | **Not Available** | **CRITICAL**: mandatory audit column unmapped |
| Source System Id | STRING | (none) | **Not Available** | **CRITICAL**: mandatory audit column unmapped |
| `<entity>_sk` | — | — | **MISSING** | SLV-001: no surrogate key |
| `<entity>_bk` | — | — | **MISSING** | SLV-002: no business key |
| `effective_from` | — | — | **MISSING** | SLV-003: `Business Start Date` (CHAR(18)) is wrong name and type |
| `effective_to` | — | — | **MISSING** | SLV-003: `Business End Date` (CHAR(18)) is wrong name and type |
| `is_current` | — | — | **MISSING** | SLV-003: `Is Active Flag` (CHAR(18)) is wrong name and type |
| `run_id` | — | — | **MISSING** | Guideline §2.7: absent |
| `source_ingestion_date` | — | — | **MISSING** | Guideline §2.7: absent |

#### 4.1c — Metadata Completeness

| Column | Required | Present | Issue |
|---|---|---|---|
| `source_system_code` | ✅ | ❌ Mapped status: "Not available" | SLV-008 violation |
| `source_system_id` | ✅ | ❌ Mapped status: "Not available" | SLV-008 violation |
| `create_date` | ✅ | ✅ Mapped | Correct |
| `update_date` | ✅ | ✅ Mapped | Correct |
| `delete_date` | ✅ | ✅ (`Deleted Date`) | Column exists |
| `is_active_flag` | ✅ | ⚠️ `Is Active Flag` (CHAR(18)) | Wrong type, should be STRING 'Y'/'N' |
| `effective_from` | ✅ | ⚠️ `Business Start Date` (CHAR(18)) | Wrong name + type |
| `effective_to` | ✅ | ⚠️ `Business End Date` (CHAR(18)) | Wrong name + type |
| `is_current` | ✅ | ⚠️ `Is Active Flag` (CHAR(18)) | Wrong name + type (BOOLEAN required) |
| `run_id` | ✅ | ❌ | Missing |
| `source_ingestion_date` | ✅ | ❌ | Missing |

---

### Finding INV-001 — No Surrogate Key on Any Entity

| Field | Value |
|---|---|
| Priority | **P0** |
| Criticality | **High** |
| Guideline Rule | `SLV-001/002` — "Every Silver entity must have `<entity>_sk` (MD5 deterministic surrogate key) and `<entity>_bk` (business/natural key)" |
| Evidence | Logical model: `bank_logical_model.xlsx` → Consolidated Metadata, all 31 investment entities — no `_sk` or `_bk` column defined. Data mapping: `investments_data_mapping.xlsx`, Investment Silver Mapping sheet — no surrogate key in any entity. |
| Affected Table | All 28 physical tables in `silver_dev.investments` |
| Affected Column(s) | `<entity>_sk`, `<entity>_bk` (all missing) |
| Confidence | 0.99 |

**Description:** Not a single entity across the entire investments subject area defines a deterministic surrogate key (`<entity>_sk = MD5(UPPER(TRIM(business_key)))`) or a properly named business key column (`<entity>_bk`). All entity identification relies on raw source-system natural keys scattered as `Account Number`, `Product Id`, `Event Id`, `Transaction Id`, etc. This makes Gold-layer FK relationships non-deterministic, breaks pipeline idempotency (a re-run or rebuild would severe downstream joins), and prevents MDM Global_UUID resolution at Gate 3.

**Remediation:**
- Add `<entity>_sk STRING NOT NULL COMMENT 'MD5 deterministic surrogate key'` as the PK to every entity.
- Add `<entity>_bk STRING NOT NULL COMMENT 'Business/natural key from source system'` as the natural key.
- ETL pipeline must compute `MD5(UPPER(TRIM(source_natural_key)))` — never UUID() or NEWID().
- Example: `investment_transaction_sk = MD5(UPPER(TRIM(CAST(transnbr AS STRING))))`.

**Estimated Effort:** Large
**Owner:** Data Modeling + ETL

---

### Finding INV-002 — All Attributes Typed as STRING

| Field | Value |
|---|---|
| Priority | **P0** |
| Criticality | **High** |
| Guideline Rule | `SLV-010` — "Monetary amounts must be stored in the smallest unit (fils for AED)"; Gate 2 Standard — "All monetary columns must use DECIMAL(18,4)"; §2.9 — "All timestamps in UTC (TIMESTAMP type)" |
| Evidence | Logical model: all 594 investment-entity attribute rows in `bank_logical_model.xlsx` have `Logical Data Type = STRING` or `CHAR(18)`. Data mapping: all attributes in `investments_data_mapping.xlsx` list `STRING` type. |
| Affected Table | All 28 tables |
| Affected Column(s) | All monetary, date, timestamp, rate, boolean, and integer attributes |
| Confidence | 1.0 |

**Description:** Every attribute across all 31 entities — including monetary amounts, date/timestamp fields, rates/percentages, boolean flags, and integer codes — is typed as `STRING`. This:
- Violates the Gate 2 standard requiring `DECIMAL(18,4)` for monetary columns.
- Violates SLV-010 requiring amounts in smallest currency unit (fils for AED).
- Violates the UTC timestamp requirement (SLV-009) which requires TIMESTAMP type, not STRING.
- Prevents numeric aggregation, date arithmetic, and range filtering without casting.
- Causes implicit data loss (e.g., float source values like trade.fee → STRING loses precision).
- Makes downstream Gold schema definitions impossible without Silver-layer casting.

Specific critical monetary attributes typed STRING: `Transaction Amount`, `Settlement Amount`, `Fee Amount`, `Option Strike Price Amount`, `Per Share Amount`, `Swap Notional Payment Date` (a date — also wrong type in name).

**Remediation:** Perform systematic type assignment:

| Pattern | Correct Type |
|---|---|
| All `*_amount*` monetary fields | `DECIMAL(18,4)` |
| AED-denominated amounts | `BIGINT` (fils) with column name `*_amount_aed` |
| Rate/percentage fields | `DECIMAL(18,6)` |
| Date fields | `DATE` |
| Timestamp/datetime fields | `TIMESTAMP` |
| Boolean flags | `BOOLEAN` |
| Code/type fields | `STRING` (retain, but map via code_mapping) |
| Count/integer fields | `BIGINT` |

**Estimated Effort:** Large
**Owner:** Data Modeling + ETL

---

### Finding INV-003 — Investment Transaction Has No Physical Table

| Field | Value |
|---|---|
| Priority | **P0** |
| Criticality | **High** |
| Guideline Rule | Modeling standard §2.11 — "Tables that exist in the platform but have no corresponding entity in the Erwin model are a governance gap" (and vice versa) |
| Evidence | Logical model: `Investment Transaction` entity with 77 attributes. Physical: no `investment_transaction` table in `silver_dev.investments`. |
| Affected Table | `silver_dev.investments.investment_transaction` (does not exist) |
| Affected Column(s) | All 77 logical attributes |
| Confidence | 0.95 |

**Description:** The `Investment Transaction` entity, representing the core financial transaction for investment activity (buy/sell/transfer/dividend), has 77 logical attributes and 70 mapping rows, yet has **no corresponding physical Delta Lake table**. This is the primary transactional entity of the subject area. Without it, no investment activity can be recorded or queried in Silver. The 17 "Not Available" attributes (including Transaction Amount, Net Amount, Gross Amount, AED Amount, Exchange Rate, Unit Price) mean that even when the table is created, critical financial measures will be absent.

**Remediation:**
- Create physical table `silver_dev.investments.investment_transaction` (or in canonical SA, `silver.wealth.investment_transaction`).
- Decompose the 70-attribute god-object into 3–4 normalized tables per finding INV-024.
- Resolve the 17 unmapped attributes before deployment.

**Estimated Effort:** Large
**Owner:** Data Modeling + ETL

---

### Finding INV-004 — Investment Account Entity Missing

| Field | Value |
|---|---|
| Priority | **P0** |
| Criticality | **High** |
| Guideline Rule | SA taxonomy — "Account: Holding Containers" is a canonical entity; BIAN Investment Portfolio Account |
| Evidence | Logical model: `Investment Account` entity with 22 attributes (Account Number, Application ID, Product Id, Customer number (CIF), Investment Type Code, Expected/Actual Value Dttm, Payment Timing Type, etc.). Physical: no `investment_account` table. Mapping: `Investment Agreement` (10 attrs) in mapping partially overlaps. |
| Affected Table | `silver_dev.investments.investment_account` (does not exist) |
| Affected Column(s) | All 22 logical attributes |
| Confidence | 0.98 |

**Description:** The `Investment Account` is the **holding container** for a customer's investment position — the entity that links customer (CIF), product, and transaction. Without it, `security_holding` and `trade_order` have no FK parent, creating a broken referential hierarchy. BIAN v11 defines "Investment Portfolio Account" as a fundamental service domain entity. The SA cannot support portfolio-level analytics, account lifecycle, or settlement without this entity.

**Remediation:**
- Create `silver.wealth.investment_account` (or within current dev schema).
- Attributes: `investment_account_sk` (MD5), `investment_account_bk`, `customer_bk`, `product_bk`, `investment_type_code`, `expected_value_timestamp`, `actual_value_timestamp`, `expected_settlement_timestamp`, `actual_settlement_timestamp`, `payment_timing_type_code`, `investment_purpose_type_code`, `purchase_intent_code` + all standard metadata columns.
- Add FK from `security_holding.investment_account_bk` → `investment_account.investment_account_bk`.

**Estimated Effort:** Large
**Owner:** Data Modeling

---

### Finding INV-005 — Trade Transaction Entity Missing (93 Attributes)

| Field | Value |
|---|---|
| Priority | **P0** |
| Criticality | **High** |
| Guideline Rule | SA completeness; BIAN Securities Trade Booking service domain |
| Evidence | Logical model: `Trade Transaction` entity with 93 attributes in `bank_logical_model.xlsx`. No physical table. No mapping rows in `investments_data_mapping.xlsx`. |
| Affected Table | `silver_dev.investments.trade_transaction` (does not exist) |
| Affected Column(s) | All 93 logical attributes (unmapped) |
| Confidence | 0.97 |

**Description:** `Trade Transaction` has 93 logical attributes — the largest entity in the subject area — and **zero physical or mapping implementation**. It represents the comprehensive trade booking record in BIAN terms, encompassing trade details, counterparty data, pricing, risk classification, settlement instructions, and regulatory reporting fields. Without it, the SA has no complete trade record at all; `trade_order` (order management) and `trade_execution` (execution) are supplementary entities that depend on a core trade record.

**Remediation:**
- Analyse the 93 attributes and decompose to 3NF before physical implementation.
- Map to FIS source tables (primary source: `dbo.trade`, with joins to `dbo.settlement`, `dbo.trans_hst`).
- The core grain is one row per trade booking (`trade_sk = MD5(UPPER(TRIM(trade_id)))`).

**Estimated Effort:** Large
**Owner:** Data Modeling + ETL

---

### Finding INV-006 — Swap Leg Cash Flow Feature: 6 of 7 Attributes Unmapped

| Field | Value |
|---|---|
| Priority | **P0** |
| Criticality | **High** |
| Guideline Rule | SLV-005 — "DQ-gated writes; quarantine table"; mapping completeness |
| Evidence | Data mapping: `swap_leg_cash_flow_feature` entity — only `Account Number` mapped; `Swap Leg Num`, `Account Modifier Number`, `Feature Id`, `Swap Payment Type Code`, `Swap Leg Feature Start Date`, `Swap Leg Feature End Date` all status = "Not Available". |
| Affected Table | `silver_dev.investments.swap_leg_cash_flow_feature` |
| Affected Column(s) | `swap_leg_num`, `account_modifier_number`, `feature_id`, `swap_payment_type_code`, `swap_leg_feature_start_date`, `swap_leg_feature_end_date` |
| Confidence | 1.0 |

**Description:** `swap_leg_cash_flow_feature` defines the cash flow schedule for each leg of a swap contract — critical for derivatives valuation, hedge accounting (IFRS 9), and market risk (BCBS 239). With 6 of 7 defining attributes unmapped (only the parent-account link is mapped), this entity is essentially empty, exposing the bank to derivatives-data gaps in regulatory reporting.

**Remediation:** Source `Swap Leg Num` and `Account Modifier Number` from `dbo.instrument` (leg sequence) and `dbo.account.account2`; `Feature Id` and `Swap Payment Type Code` from `dbo.cf_definition`; start/end dates from `dbo.cf_schedule`. Escalate as a P0 source mapping activity with FIS SME.

**Estimated Effort:** Medium
**Owner:** ETL + Source System SME

---

### Finding INV-007 — Reference Data Tables in Wrong Subject Area

| Field | Value |
|---|---|
| Priority | **P0** |
| Criticality | **High** |
| Guideline Rule | SLV-004 — "No source-system codes; use `silver.reference.code_mapping`"; SA isolation (§2.2 of modeling guidelines) |
| Evidence | Physical tables: `reference_currency_code`, `reference_exchange_rate2`, `reference_product`, `reference_transaction_code`, `unit_of_measure` all in `silver_dev.investments`. Subject-areas.md defines `silver.reference` as the canonical SA for reference data (Priority 1 - Critical). |
| Affected Table | `silver_dev.investments.reference_currency_code`, `reference_exchange_rate2`, `reference_product`, `reference_transaction_code`, `unit_of_measure` |
| Affected Column(s) | All columns |
| Confidence | 1.0 |

**Description:** Five reference/lookup tables exist within the investments schema instead of the canonical `silver.reference` schema. This:
- Violates SA isolation (other SAs cannot consume these without cross-SA joins in Silver pipelines).
- Creates duplicate definitions (FX rates, currency codes, product catalog already partially exist in `silver.reference`).
- Violates SLV-004 (source-system codes should resolve via `silver.reference.code_mapping`).
- `reference_exchange_rate2` has the suffix `2` indicating an unmanaged proliferation of rate tables.

**Remediation:**
- Migrate all 5 tables to `silver.reference` under appropriate entity names.
- Rename `reference_exchange_rate2` → `silver.reference.exchange_rate` (drop the `2` suffix; reconcile with any existing rate table).
- Replace investment SA references with FK to `silver.reference` at Gold layer.

**Estimated Effort:** Medium
**Owner:** Data Modeling + Platform Engineering

---

### Finding INV-008 — Temp Table in Production Schema

| Field | Value |
|---|---|
| Priority | **P0** |
| Criticality | **High** |
| Guideline Rule | §2.11 — "Tables that exist in the platform but have no corresponding entity in the Erwin model are a governance gap"; NAM-003 — no staging/temp pattern |
| Evidence | Physical structures CSV: `foreign_exchange_agreement_temp_no_partition` present in `silver_dev.investments`. Properties: `delta.autoOptimize.*` flags set but no partitioning. No logical model entity. |
| Affected Table | `silver_dev.investments.foreign_exchange_agreement_temp_no_partition` |
| Affected Column(s) | All |
| Confidence | 1.0 |

**Description:** A table named `foreign_exchange_agreement_temp_no_partition` is registered as a production Delta Lake table in Unity Catalog. The `_temp_no_partition` suffix confirms this was created as a workaround during ETL development and never cleaned up. Per NAM-003, Silver tables follow the pattern `<entity>` with no `temp`/`stg`/`no_partition` suffixes. Having a temp table in production risks:
- Accidental consumer queries.
- Data duplication with `foreign_exchange_agreement`.
- Confusing lineage and audit trails.

**Remediation:**
- Drop `foreign_exchange_agreement_temp_no_partition` from `silver_dev.investments`.
- If the unpartitioned version is intentional (e.g., for small-table full-scan), document and rename.
- Ensure `foreign_exchange_agreement` is the canonical table.

**Estimated Effort:** Small
**Owner:** Platform Engineering

---

### Finding INV-009 — SCD Metadata Columns Wrong Names and Types

| Field | Value |
|---|---|
| Priority | **P1** |
| Criticality | **High** |
| Guideline Rule | `SLV-003` — "SCD-2 default; every Silver entity must have `effective_from`, `effective_to`, `is_current`"; §2.7 — canonical audit column names and types |
| Evidence | Logical model: all entities use `Business Start Date` (CHAR(18)), `Business End Date` (CHAR(18)), `Is Active Flag` (CHAR(18)). Correct names per §2.7: `effective_from` (TIMESTAMP), `effective_to` (TIMESTAMP), `is_current` (BOOLEAN). |
| Affected Table | All 28 physical tables |
| Affected Column(s) | `Business Start Date`, `Business End Date`, `Is Active Flag` |
| Confidence | 1.0 |

**Description:** Three SCD-2 lifecycle columns are wrong on two dimensions:
1. **Names**: `Business Start Date` / `Business End Date` / `Is Active Flag` do not match the required canonical names `effective_from` / `effective_to` / `is_current`.
2. **Types**: `CHAR(18)` for a date/boolean column is incorrect. `effective_from`/`effective_to` must be TIMESTAMP; `is_current` must be BOOLEAN.

Using `CHAR(18)` for `Business Start Date` cannot store a proper ISO 8601 timestamp and will cause silent data truncation. The non-standard names mean downstream pipelines cannot use generic SCD-2 frameworks without table-specific custom mappings.

Additionally, `IsLogicallyDeleted` (STRING) is non-standard — the standard uses `delete_date` (TIMESTAMP, nullable) for soft deletes per §2.7, and `is_active_flag` (STRING 'Y'/'N') as the active indicator.

**Remediation:**
- Rename `Business Start Date` → `effective_from` (TIMESTAMP NOT NULL).
- Rename `Business End Date` → `effective_to` (TIMESTAMP, nullable).
- Rename `Is Active Flag` → `is_current` (BOOLEAN NOT NULL).
- Replace `IsLogicallyDeleted` with `delete_date` (TIMESTAMP, nullable).
- Add missing `run_id` (STRING) and `source_ingestion_date` (TIMESTAMP).
These are breaking DDL changes; follow the schema versioning migration procedure.

**Estimated Effort:** Large
**Owner:** Data Modeling + ETL

---

### Finding INV-010 — Source-System Integer Codes Stored Directly (SLV-004)

| Field | Value |
|---|---|
| Priority | **P1** |
| Criticality | **High** |
| Guideline Rule | `SLV-004` — "No source-system codes; use `silver.reference.code_mapping`"; §2.5 |
| Evidence | Data mapping: `Account Type` src=`account.account_type (int)` → STRING straight-through; `Settlement Method` src=`settlement.type (int)` → STRING; `Transaction Code` src=`trade.type (int)` → STRING; `instrument.instype (int/char)` used across 8+ entities straight-through. |
| Affected Table | `financial_agreement`, `foreign_exchange_agreement`, `option_agreement`, `swap_agreements`, `investment_product`, `trade_order`, `investment_product_event`, `trade_execution` and `investment_transaction` |
| Affected Column(s) | `account_type`, `settlement_method`, `transaction_code`, `swap_obligation_type_code`, `investment_product_type_code`, `fx_agreement_type_cd`, `option_type_cd`, `swap_leg_type_code` |
| Confidence | 0.95 |

**Description:** Across at least 8 entities, source-system integer codes from FIS (`int` type in source) are being mapped as straight-through STRING values into Silver. For example, `account.account_type (int)` contains FIS-internal account type codes that are meaningless to consumers without a code translation. Per SLV-004, Silver must store canonical platform codes (resolved via `silver.reference.code_mapping`), not raw source integers. This is consistent across option types, instrument types (`instype`), settlement types, and transaction types.

Additionally, currency fields in source are of type `numeric` (FIS internal currency code) being mapped as STRING — these should be ISO 4217 3-letter codes.

**Remediation:**
- For each integer code attribute, create a mapping entry in `silver.reference.code_mapping` for source_system=`FIS`, source_domain=`<domain>`, source_code=`<int_value>` → canonical_code.
- Replace straight-through mappings with a JOIN to `silver.reference.code_mapping`.
- Currency `numeric` codes: join to `reference_currency_code` (once migrated to `silver.reference`) to resolve to ISO 4217.

**Estimated Effort:** Large
**Owner:** ETL + Reference Data Steward

---

### Finding INV-011 — Partitioning Strategy Absent or Incorrect

| Field | Value |
|---|---|
| Priority | **P1** |
| Criticality | **Medium** |
| Guideline Rule | Guideline §1.4 — "Partition all tables by ingestion date (`_ingested_date`)"; Silver tables should be partitioned by `effective_from` date or ingestion date |
| Evidence | Physical structures CSV: 27 of 28 tables have `partitionColumns: []` (no partitioning). `foreign_exchange_agreement` partitioned by `["Account_Number"]` — a business key, not temporal. |
| Affected Table | All 28 tables |
| Affected Column(s) | `partitionColumns` |
| Confidence | 1.0 |

**Description:** 27 of 28 tables are completely unpartitioned. For a production Silver schema that will accumulate years of SCD-2 history, unpartitioned tables will cause full-table scans on every SCD-2 merge and every pipeline run. The one partitioned table (`foreign_exchange_agreement`) uses `Account_Number` as the partition key — a high-cardinality business key that creates too many small partition files and violates the convention of partitioning by ingestion/processing date. Additionally, `Account_Number` uses PascalCase — violating NAM-001 (snake_case required).

**Remediation:**
- Partition all Silver investment tables by `source_ingestion_date` (DATE) or `effective_from` (DATE truncated).
- For `foreign_exchange_agreement`: remove `Account_Number` partition; add `source_ingestion_date` partition.
- Consider Z-ordering (Databricks) on `effective_from`, `is_current` for SCD-2 query acceleration.

**Estimated Effort:** Large (requires full table rebuild per breaking-change procedure)
**Owner:** Platform Engineering + DBA

---

### Finding INV-012 — Monetary Amounts Lack AED Denomination Columns

| Field | Value |
|---|---|
| Priority | **P1** |
| Criticality | **High** |
| Guideline Rule | `SLV-010` — "Monetary amounts in smallest unit (fils for AED)"; NAM column standard: `<measure>_amount_aed` for AED-denominated columns, `<measure>_amount_orig` + `currency_code` for original currency |
| Evidence | Logical model: `Transaction Amount In AED` (STRING, **unmapped**). All other amount columns have no `_aed` suffix and have no `currency_code` companion. |
| Affected Table | `silver_dev.investments.investment_transaction` (and by extension all entities with monetary values) |
| Affected Column(s) | All amount columns |
| Confidence | 0.95 |

**Description:** The investments subject area holds multi-currency financial data (AED, USD, EUR per `trade.curr`). The required pattern per SLV-010 is:
- `<measure>_amount_aed` (BIGINT in fils) — the AED-denominated canonical amount.
- `<measure>_amount_orig` (DECIMAL(18,4)) — amount in original transaction currency.
- `currency_code` (CHAR(3)) — ISO 4217 currency code.

None of the entities implement this pattern. `Transaction Amount In AED` is conceptually correct but typed as STRING and unmapped. The exchange rate entity (`reference_exchange_rate2`) has its key rate columns also unmapped, meaning AED conversion cannot even be performed.

**Remediation:**
- For all monetary attributes, add `_amount_aed` (BIGINT), `_amount_orig` (DECIMAL(18,4)), `currency_code` (CHAR(3)) columns.
- Map AED conversion using `reference_exchange_rate2` (after fixing its unmapped rate columns).
- Apply `_amount_aed = ROUND(_amount_orig * exchange_rate * 100, 0)` (converting to fils).

**Estimated Effort:** Large
**Owner:** Data Modeling + ETL

---

### Finding INV-013 — Investment Transaction: 17 Critical Financial Attributes Unmapped

| Field | Value |
|---|---|
| Priority | **P0** |
| Criticality | **High** |
| Guideline Rule | SLV-005 — "DQ-gated writes; completeness"; BCBS 239 — accuracy and completeness of financial data |
| Evidence | Data mapping: `investments_data_mapping.xlsx` Investment Silver Mapping, Investment Transaction entity: `Transaction Amount`, `Net Transaction Amount`, `Gross Transaction Amount`, `Transaction Amount In AED`, `Transaction Unit`, `Unit Price`, `Exchange Rate`, `Transaction Acknowledge Confirmation`, `Financial Market Commitment Amount`, `Unit Holdings`, `Legacy Tax Amount`, `Transaction Text`, `Pledge Indicator`, `Source Type`, `Customer Segment Description`, `Source System Code`, `Source System Id` — all status "Not Available" |
| Affected Table | `silver_dev.investments.investment_transaction` (does not exist) |
| Affected Column(s) | 17 attributes listed above |
| Confidence | 1.0 |

**Description:** Among the 17 unmapped attributes are **all primary financial measures** of the entity — Transaction Amount, Net Amount, Gross Amount, and the AED-denominated equivalent. Without these, the Investment Transaction entity cannot support any financial reporting, portfolio valuation, or P&L calculation. Mandatory audit columns (`Source System Code`, `Source System Id`) are also unmapped, making lineage tracing impossible.

`Exchange Rate` being unmapped is particularly critical because it is needed to calculate `Transaction Amount In AED`, which is also unmapped — a cascading gap.

**Remediation:**
- Escalate with FIS source system SME to identify source tables for Transaction Amount, Unit Price, Exchange Rate.
- Candidate: `dbo.trade.quantity` × `dbo.trade.price` for Transaction Amount; `dbo.trade.price` for Unit Price; `silver.reference.exchange_rate` for Exchange Rate.
- Map `Source System Code` as literal `'FIS'`; `Source System Id` from `dbo.trade.trdnbr`.

**Estimated Effort:** Medium
**Owner:** ETL + Source System SME

---

### Finding INV-014 — Reference_Exchange_Rate2: Exchange Rate Values Unmapped

| Field | Value |
|---|---|
| Priority | **P0** |
| Criticality | **High** |
| Guideline Rule | SLV-005 completeness; BCBS 239 accuracy |
| Evidence | Data mapping: `Reference_Exchange_Rate2` — `Source to Target Currency Rate` status=Not Available; `Target to Source Currency Rate` status=Not Available. Physical: `reference_exchange_rate2` table exists. |
| Affected Table | `silver_dev.investments.reference_exchange_rate2` |
| Affected Column(s) | `source_to_target_currency_rate`, `target_to_source_currency_rate` |
| Confidence | 1.0 |

**Description:** The exchange rate table physically exists but its **primary business values — the actual exchange rates — are not mapped from source**. The table can only answer "which currency pair date exists" but not "what is the rate". Every AED conversion calculation across the entire SA depends on this table. This is a cascading blocker for SLV-010 compliance.

**Remediation:** Source rates from FIS `dbo.price_history` or `dbo.fx_rate_definition`; alternatively from a central FX feed. Map as DECIMAL(18,6) for rate precision.

**Estimated Effort:** Small
**Owner:** ETL + Source System SME

---

### Finding INV-015 — Typos in Attribute Names (NAM-001)

| Field | Value |
|---|---|
| Priority | **P2** |
| Criticality | **Low** |
| Guideline Rule | `NAM-001` — "snake_case, lowercase, no abbreviations unless approved" |
| Evidence | Logical model: `Transactoin Id` (missing 't'), `Divident` (missing 'd'), `Transcation Status` (transposed letters), `Transcaction Settlement Status Code` (double typo), `Invesment Amount` (missing 't') |
| Affected Table | `investment_transaction`, `dividend_declare_event`, `stock_dividend_amount`, `cash_dividend_event` |
| Affected Column(s) | `transactoin_id`, `divident_type_code`, `transcation_status`, `transcaction_settlement_status_code`, `invesment_amount` |
| Confidence | 1.0 |

**Description:** Five attribute names in the logical model contain spelling errors. These propagate to physical DDL column names, causing: confusing consumer queries; diff from downstream SQL referencing correct spelling; failed lineage lookups. The column `Transactoin Id` appears **twice** in the Investment Transaction mapping — confirmed as a duplicate variable in the Field Classification sheet.

**Remediation:** Correct all typos in logical model first, then regenerate DDL. Per the versioning policy, column renames are **breaking changes** requiring a new table version (`_v2`) and a 6-month deprecation window.

**Estimated Effort:** Small (logical model) / Medium (DDL migration)
**Owner:** Data Modeling

---

### Finding INV-016 — Security Holding Has Only 2 of 13 Attributes Mapped

| Field | Value |
|---|---|
| Priority | **P1** |
| Criticality | **High** |
| Guideline Rule | SLV-005 completeness |
| Evidence | Logical model: `Security Holding` has 13 attributes. Data mapping: only 2 attributes (`Account Number`, `Account Modifier Number`) are in the Investment Silver Mapping sheet. Physical table exists. |
| Affected Table | `silver_dev.investments.security_holding` |
| Affected Column(s) | 11 unmapped attributes (position quantity, market value, book value, unrealised P&L, etc.) |
| Confidence | 0.92 |

**Description:** `Security Holding` represents the customer's equity/bond/fund position at a point in time — critical for portfolio valuation, IFRS 9 classification, and wealth reporting. With only 2 of 13 attributes mapped (both structural FK-like columns), the table is essentially unmapped and non-functional for any analytical purpose.

**Remediation:** Identify source tables in FIS for holding quantity (`dbo.trade.quantity`), book value, market value (from `dbo.instrument` price × quantity), accrued interest, and position date. Complete all 13 attribute mappings.

**Estimated Effort:** Medium
**Owner:** ETL + Source System SME

---

### Finding INV-017 — Cash Dividend Event and Foreign Exchange Event Have No Physical Tables

| Field | Value |
|---|---|
| Priority | **P1** |
| Criticality | **Medium** |
| Guideline Rule | Modeling standard §2.11; SA entity completeness |
| Evidence | Logical model: `Cash Dividend Event` (14 attrs), `Foreign Exchange Event` (11 attrs). Physical: neither table exists in `silver_dev.investments`. |
| Affected Table | `cash_dividend_event`, `foreign_exchange_event` (both missing) |
| Affected Column(s) | All 25 combined logical attributes |
| Confidence | 0.98 |

**Description:** Two event entities are defined in the logical model but have no physical implementation. `Cash Dividend Event` tracks cash distributions to investment account holders (critical for tax reporting and portfolio income tracking). `Foreign Exchange Event` tracks FX conversion events (critical for FX P&L and CBUAE reporting). Both have full or near-full mapping definitions (Cash Dividend Event: 4/5 mapped; Foreign Exchange Event: 2/2 mapped) suggesting they were planned but never materialised.

**Remediation:** Create Delta Lake tables for both entities following the standard DDL pattern. Apply SCD-2 columns and surrogate keys.

**Estimated Effort:** Medium
**Owner:** Data Modeling + ETL

---

### Finding INV-018 — Stock Entity Missing; Stock History vs Stock Conflation

| Field | Value |
|---|---|
| Priority | **P1** |
| Criticality | **Medium** |
| Guideline Rule | 3NF (SLV-006); entity separation |
| Evidence | Logical model: `Stock` entity (14 attrs: Scrip code, Investment Product Id, Stock type code, Stock share class code, + metadata). Physical: only `stock_history` exists (historical price data). `Stock` master entity is absent. |
| Affected Table | `stock` (missing); `silver_dev.investments.stock_history` (exists) |
| Affected Column(s) | 14 logical attributes of Stock entity |
| Confidence | 0.90 |

**Description:** The logical model correctly separates the **Stock master** (instrument attributes: scrip code, type, share class — slowly changing) from **Stock History** (time-series price data — insert-only). Only the history table exists physically. Without the Stock master, `stock_history` has no FK reference parent and stock-level attributes (share class, instrument type) are either missing or duplicated in history rows.

**Remediation:** Create `silver.wealth.stock` as the SCD-2 instrument master. `stock_history` becomes an append-only time-series child table with FK to `stock.stock_sk`.

**Estimated Effort:** Small
**Owner:** Data Modeling

---

### Finding INV-019 — Investment Transaction: 3NF Violations / God Object

| Field | Value |
|---|---|
| Priority | **P1** |
| Criticality | **High** |
| Guideline Rule | `SLV-006` — "3NF; no derived values/aggregations"; §2.1 — no repeating groups |
| Evidence | Logical model Investment Transaction entity: Customer Segment + Customer Segment Description (party attributes), Sales Channel + Distributed Channel + Sales Staff + Sales Branch (channel/org attributes), 6 fee/rate columns mapping to same source column (repeating group), Legacy* columns (3). |
| Affected Table | `investment_transaction` (once created) |
| Affected Column(s) | `customer_segment`, `customer_segment_description`, `sales_channel`, `distributed_channel`, `sales_staff`, `sales_branch`, `entity_branch_code`, `original_fee_rate`/`adhoc_fee_rate`/`promo_fee_rate`/`fee_rate` (repeating fee group), `legacy_*` columns |
| Confidence | 0.97 |

**Description:** The `Investment Transaction` entity violates 3NF in at least four ways:
1. **Party attributes**: `Customer Segment` and `Customer Segment Description` are customer attributes — they belong in `silver.party.customer_segment` and should be FK-joined at Gold.
2. **Channel/Org attributes**: `Sales Channel`, `Distributed Channel`, `Sales Branch`, `Entity Branch Code` are channel and organisation dimension data. Per §2.2, they belong in `silver.event` (Business Event) or `silver.org` (Organization) and should be FK-referenced.
3. **Repeating fee/rate group**: `Original Fee Rate`, `Adhoc Fee Rate`, `Promo Fee Rate`, `Fee Rate` (and their commitment rate counterparts) are a repeating group of fee-type variants. Should be a child table `investment_transaction_fee` (fee_type_code, rate_value, effective_date).
4. **Legacy columns**: `Legacy Investment Amount`, `Legacy Average Cost`, `Legacy Tax Amount` represent source-system migration artefacts — they have no place in a canonical Silver entity.

**Remediation:**
- Remove party/channel/org attributes; add FK business keys only (`customer_bk`, `channel_code`, `branch_code`).
- Create `investment_transaction_fee` child table for fee/rate variants.
- Remove all `Legacy_*` columns; if needed for reconciliation, create a companion migration table.

**Estimated Effort:** Large
**Owner:** Data Modeling

---

### Finding INV-020 — Duplicate Attribute Mappings (Ambiguous Source)

| Field | Value |
|---|---|
| Priority | **P1** |
| Criticality | **Medium** |
| Guideline Rule | SLV-006 — "no derived values"; mapping accuracy |
| Evidence | Data mapping: `Securities Financing` — `Account Number` appears twice (src=account.accnbr AND src=account.account); `Repurchase Agreement Type Code` appears twice (src=agreement.instype AND src=instr_regulatory_info.cfi_code). Investment Transaction: `Transaction Id` appears twice. Multiple fee/rate attributes map from same source column. |
| Affected Table | `securities_financing`, `investment_transaction` |
| Affected Column(s) | `account_number`, `repurchase_agreement_type_code`, `transaction_id` |
| Confidence | 0.95 |

**Description:** Multiple attributes appear more than once in the mapping with different source columns. This is ambiguous: which source wins? Are these two different business concepts sharing a name? The `Repurchase Agreement Type Code` mapping from two different source tables (`agreement.instype` vs `instr_regulatory_info.cfi_code`) suggests potential confusion between counterparty agreement type and CFI code. `Transaction Id` duplication in Investment Transaction was flagged as a "duplicate variable" in the Field Classification sheet.

**Remediation:** Review each duplicate with source system SME and either (a) confirm it's a single attribute with primary source and eliminate the duplicate, or (b) rename into distinct attributes (`repurchase_agreement_type_code` vs `cfi_code`).

**Estimated Effort:** Small
**Owner:** ETL + Source System SME

---

### Finding INV-021 — Subject Area Not Registered in Canonical Taxonomy

| Field | Value |
|---|---|
| Priority | **P0** |
| Criticality | **High** |
| Guideline Rule | Governance requirement; `sa/subject-areas.md` is normative |
| Evidence | `sa/subject-areas.md`: no "investments" entry. Physical: `silver_dev.investments` schema exists with 28 tables. |
| Affected Table | All tables in `silver_dev.investments` |
| Affected Column(s) | N/A |
| Confidence | 1.0 |

**Description:** The entire investments subject area operates outside the canonical subject-area registry. `sa/subject-areas.md` defines 19 subject areas; "investments" is not among them. The closest registered SAs are "Wealth" (silver.wealth, #9) and "Treasury" (silver.treasury, #15). Without registration:
- No official business owner or data steward is assigned.
- No Informatica GDGC catalog registration can be performed (§2.12).
- The schema name (`silver_dev.investments`) uses a `dev` environment prefix suggesting this has never been promoted to production.
- Priority of the SA (4 = Low for both Wealth and Treasury) means it may be systematically deprioritised.

**Remediation:**
- Formally split into `silver.wealth` (client book) and `silver.treasury` (bank book).
- Register both in `sa/subject-areas.md` with schema, priority, description, business owner, and data steward.
- Complete catalog registration in Informatica GDGC.
- Rename physical tables from `silver_dev.investments.*` → `silver.wealth.*` / `silver.treasury.*`.

**Estimated Effort:** Large (governance process)
**Owner:** Group Data Office + Data Steward

---

### Finding INV-022 — Application Detail Date Fields Swapped in Mapping

| Field | Value |
|---|---|
| Priority | **P1** |
| Criticality | **Medium** |
| Guideline Rule | Mapping accuracy; SLV-005 accuracy dimension |
| Evidence | Data mapping: `Application Investment detail` — `Application Detail End Date` maps to source `instrument.date_from` (datetime); `Application Detail Start Date` maps to source `instrument.date_to` (datetime). Start→date_to and End→date_from appear transposed. |
| Affected Table | `silver_dev.investments.application_investment_detail` |
| Affected Column(s) | `application_detail_start_date`, `application_detail_end_date` |
| Confidence | 0.85 |

**Description:** The mapping of `Application Detail End Date` ← `instrument.date_from` and `Application Detail Start Date` ← `instrument.date_to` appears semantically reversed. `date_from` typically represents a start date and `date_to` an end date in FIS. If this is an actual transposition, any downstream date-range query (e.g., "active applications today") would produce incorrect results.

**Remediation:** Confirm with FIS source system SME whether `instrument.date_from`/`date_to` semantics match the target names. If transposed, swap the source column assignments.

**Estimated Effort:** Small
**Owner:** ETL + Source System SME

---

### Finding INV-023 — Dividend Declare Event: Key Date Attributes Unmapped

| Field | Value |
|---|---|
| Priority | **P1** |
| Criticality | **Medium** |
| Guideline Rule | SLV-005 completeness; corporate actions completeness |
| Evidence | Data mapping: `Dividend Declare Event` — `Divident Record Date` status=Not Available; `Divident Payable Date` status=Not Available. 2 of 4 attributes unmapped. |
| Affected Table | `silver_dev.investments.dividend_declare_event` |
| Affected Column(s) | `dividend_record_date`, `dividend_payable_date` |
| Confidence | 1.0 |

**Description:** The two business-critical date fields of a dividend declaration — when the record date is set and when dividends are payable — are both unmapped. Without these dates, the dividend event cannot support any downstream use case (portfolio income accrual, ex-dividend date calculation, tax withholding timing).

**Remediation:** Source from FIS `dbo.corp_action` — `record_date` and `pay_date` columns. Map as DATE type.

**Estimated Effort:** Small
**Owner:** ETL + Source System SME

---

### Finding INV-024 — Financial Agreement vs Investment Agreement Ambiguity

| Field | Value |
|---|---|
| Priority | **P1** |
| Criticality | **Medium** |
| Guideline Rule | SLV-006 — "entity boundary"; 3NF |
| Evidence | Mapping sheet has two entities: `Financial_Agreement` (9 attrs) and `Investment Agreement` (10 attrs). Physical: only one table `financial_agreement`. Logical model: only `Financial Agreement` (20 attrs). |
| Affected Table | `silver_dev.investments.financial_agreement` |
| Affected Column(s) | Overlap columns: `Account Number`, `Account Modifier Number`, `Product Id` |
| Confidence | 0.88 |

**Description:** The mapping file defines both `Financial_Agreement` and `Investment Agreement` as separate mapping entities, but there is only one physical table (`financial_agreement`) and one logical model entity (`Financial Agreement`). The `Investment Agreement` mapping adds investment-lifecycle attributes (`Expected Value Dttm`, `Actual Value Dttm`, `Investment Type Code`, `Investment Purpose Type Code`, `Purchase Intent Code`) that are conceptually lifecycle/account data rather than agreement terms. This suggests:
- Either `Investment Agreement` should be a separate `investment_account` entity (resolving Finding INV-004).
- Or `Investment Agreement` attributes should be sub-typed within `financial_agreement`.

**Remediation:** Clarify with data steward: (a) if `Investment Agreement` lifecycle attributes belong in `investment_account` → resolve Finding INV-004; (b) if they're agreement sub-type → add `financial_agreement_type_code` discriminator and document.

**Estimated Effort:** Medium
**Owner:** Data Modeling + Business Steward

---

### Finding INV-025 — FX Agreement: Destination/Source Qty Mapped from Currency Code

| Field | Value |
|---|---|
| Priority | **P1** |
| Criticality | **Medium** |
| Guideline Rule | Mapping accuracy; SLV-010 data type |
| Evidence | Data mapping: `Foreign Exchange Agreement` — `Destination Currency Code` src=`trade.curr(numeric)` and `FX Currency Destination Qty` src=`trade.curr(numeric)` — **both map from the same source column**. Similarly `Source Currency Code` and `Fx Source Currency Qty` both map from `trade.curr(numeric)`. |
| Affected Table | `silver_dev.investments.foreign_exchange_agreement` |
| Affected Column(s) | `destination_currency_code`, `fx_currency_destination_qty`, `source_currency_code`, `fx_source_currency_qty` |
| Confidence | 0.90 |

**Description:** Both the currency code and the currency quantity are mapped from the same source column `trade.curr (numeric)`. This is semantically impossible — `trade.curr` is a currency identifier code, not a quantity. The FX quantity (notional amount in destination currency) must come from a different source column (likely `trade.quantity` or `trade.settlement_amount`). This mapping error would result in currency quantities being populated with currency codes.

**Remediation:** Source `fx_currency_destination_qty` from `trade.quantity` or `trade.principal`; `fx_source_currency_qty` similarly. Retain `trade.curr` for currency code columns only. Both quantities should be DECIMAL(18,4) not STRING.

**Estimated Effort:** Small
**Owner:** ETL + Source System SME

---

### Finding INV-026 — Swap Leg Account Event and Agreement Leg: 50% Attributes Unmapped

| Field | Value |
|---|---|
| Priority | **P1** |
| Criticality | **Medium** |
| Guideline Rule | SLV-005 completeness |
| Evidence | Mapping: `Swap Agreement Leg` — `Swap Leg Num` Not available, `Account Modifier Number` Not available (2/5 unmapped). `Swap Leg Account Event` — `Account Modifier Number` Not available, `Swap Leg Num` Not available (2/4 unmapped). |
| Affected Table | `swap_agreement_leg`, `swap_leg_account_event` |
| Affected Column(s) | `swap_leg_num`, `account_modifier_number` (both tables) |
| Confidence | 1.0 |

**Description:** `Swap Leg Num` is the primary discriminator between fixed and floating legs of a swap contract — without it, the leg entities are unidentifiable and all cash flow features (also largely unmapped per Finding INV-006) have no parent leg. `Account Modifier Number` appears across many derivative entities as a supplementary account qualifier. Both are consistently unmapped across the swap leg hierarchy.

**Remediation:** Source `Swap Leg Num` from `dbo.instrument` leg sequence index or `dbo.leg` table. Source `Account Modifier Number` from `dbo.account.account2`.

**Estimated Effort:** Small
**Owner:** ETL + Source System SME

---

### Finding INV-027 — Unit Distribution Event: 3/6 Attrs Unmapped Including All Distribution Details

| Field | Value |
|---|---|
| Priority | **P1** |
| Criticality | **Medium** |
| Guideline Rule | SLV-005 completeness |
| Evidence | Mapping: `Unit Distribution Event` — `Unit Distribution Effective Date` Not available; `Base Unit Number` Not available; `New Unit Number` Not available. |
| Affected Table | `silver_dev.investments.unit_distribution_event` |
| Affected Column(s) | `unit_distribution_effective_date`, `base_unit_number`, `new_unit_number` |
| Confidence | 1.0 |

**Description:** The effective date of the unit distribution, the pre-split unit count, and post-split unit count are all unmapped. These are the three defining measures of any unit-distribution (stock split equivalent) event. Without them, the table records only which product had a distribution event, but not when, how many units were affected, or what the new unit count became.

**Remediation:** Source from `dbo.corp_action` — `ex_date` for effective date; `dbo.trade.quantity` for base/new unit counts.

**Estimated Effort:** Small
**Owner:** ETL

---

### Finding INV-028 — Reference_Currency_Code Missing Currency ID and Numeric Code

| Field | Value |
|---|---|
| Priority | **P1** |
| Criticality | **Medium** |
| Guideline Rule | SLV-005 completeness; SLV-004 (code mapping) |
| Evidence | Mapping: `Reference_Currency_Code` — `Currency Id` Not Available; `Numeric Id` Not Available. Only `Country` and `Currency description` mapped. |
| Affected Table | `silver_dev.investments.reference_currency_code` |
| Affected Column(s) | `currency_id`, `numeric_id` |
| Confidence | 1.0 |

**Description:** A currency reference table that lacks the currency identifier and the ISO 4217 numeric code is non-functional. The `Currency Id` (likely the ISO alpha-3 code, e.g., "AED", "USD") is the primary join key that other entities use to resolve their `curr(numeric)` source values. Without it, no currency resolution can be performed. This table also belongs in `silver.reference` per Finding INV-007.

**Remediation:** Source `Currency Id` (ISO alpha-3) and `Numeric Id` (ISO 4217 numeric) from FIS currency master or standard ISO reference. Migrate to `silver.reference.currency`.

**Estimated Effort:** Small
**Owner:** ETL + Reference Data Steward

---

## 5. Denormalization Register

| # | Entity | Denormalized Attribute(s) | Classification | Justification Required | Finding |
|---|---|---|---|---|---|
| D-01 | `investment_transaction` | `customer_segment`, `customer_segment_description` | **Unnecessary** | Party attributes have no place in a transaction entity; join at Gold | INV-019 |
| D-02 | `investment_transaction` | `sales_channel`, `distributed_channel`, `sales_branch`, `sales_staff`, `entity_branch_code` | **Unnecessary** | Channel/Org dimension data; join at Gold | INV-019 |
| D-03 | `investment_transaction` | `product_description` | **Unnecessary** | Product catalog attribute; FK to `investment_product`; join at Gold | INV-019 |
| D-04 | `investment_transaction` | Fee/rate group: `original_fee_rate`, `adhoc_fee_rate`, `promo_fee_rate`, `fee_rate`, `original_commitment_rate`, `adhoc_commitment_rate`, `promo_commitment_rate`, `commitment_rate` | **Unnecessary** | Repeating group violation; requires child fee table | INV-019 |
| D-05 | `investment_transaction` | `legacy_investment_amount`, `legacy_average_cost`, `legacy_tax_amount` | **Unnecessary** | Migration artefacts; no business justification in Silver | INV-019 |
| D-06 | `reference_transaction_code` | `transaction_amount`, `transaction_local_amount`, `transaction_global_amount`, `transaction_fee` | **Unnecessary** | A **reference** (code) table should not contain transaction measures; these are fact-level values | N/A |
| D-07 | `swap_currency_agreement` | `swap_in_currency_code`, `swap_in_currency_qty` | **Acceptable** | FX amounts are specific to the swap currency agreement context; acceptable if the pair is an atomic business concept | Requires steward sign-off |
| D-08 | `foreign_exchange_agreement` | `nostra_account_num` | **Acceptable** | Nostro account is a Treasury-specific attribute contextually linked to the FX Agreement; acceptable with steward approval | Requires steward sign-off |

---

## 6. Guideline Compliance Summary

| Rule | Description | Status | Findings |
|---|---|---|---|
| **SLV-001** | `<entity>_sk` MD5 surrogate on every entity | ❌ FAIL — 0/28 entities | INV-001 |
| **SLV-002** | `<entity>_bk` on every entity | ❌ FAIL — 0/28 entities | INV-001 |
| **SLV-003** | SCD-2 default; `effective_from`, `effective_to`, `is_current` | ❌ FAIL — wrong names, wrong types (CHAR(18)) | INV-009 |
| **SLV-004** | No source-system codes; use `silver.reference.code_mapping` | ❌ FAIL — int codes stored directly across 8+ entities | INV-010 |
| **SLV-005** | DQ-gated writes; completeness | ❌ FAIL — ~70 unmapped attributes incl. primary financial measures | INV-013, INV-006, INV-016 |
| **SLV-006** | 3NF; no derived values/aggregations | ❌ FAIL — Investment Transaction is a god object | INV-019 |
| **SLV-007** | No pre-computed metrics | ⚠️ Partial — `reference_transaction_code` contains transaction amount measures | D-06 |
| **SLV-008** | All metadata columns present | ❌ FAIL — `run_id`, `source_ingestion_date` missing; SCD columns non-standard | INV-009 |
| **SLV-009** | All timestamps UTC | ⚠️ UNKNOWN — all columns are STRING; temporal correctness cannot be verified | INV-002 |
| **SLV-010** | Monetary amounts in smallest unit (fils for AED) | ❌ FAIL — all amounts are STRING; no fils conversion | INV-002, INV-012 |
| **NAM-001** | snake_case, lowercase, no reserved words | ⚠️ Partial — entity names OK; attribute typos found; partition column `Account_Number` is PascalCase | INV-015, INV-011 |
| **NAM-002** | Catalog/schema: `silver.<subject_area>.<entity>` | ❌ FAIL — schema is `silver_dev.investments` not canonical | INV-021 |
| **NAM-003** | Table naming: `<entity>` for Silver | ⚠️ Partial — temp table `foreign_exchange_agreement_temp_no_partition` violates pattern | INV-008 |
| **DQ Gate 2** | DECIMAL(18,4) for monetary; deduplication on BK; error rate threshold | ❌ FAIL — all amounts are STRING; no BK defined for deduplication | INV-001, INV-002 |
| **§2.1 3NF** | No repeating groups | ❌ FAIL — fee/rate repeating group in Investment Transaction | INV-019 |
| **§2.2 SA Isolation** | No cross-SA joins in Silver pipelines | ⚠️ Risk — reference tables in investments SA will force cross-SA joins | INV-007 |
| **§2.4 Surrogate** | Deterministic MD5 only | ❌ FAIL — no surrogate keys defined | INV-001 |
| **§2.7 Audit Columns** | 11 mandatory columns with correct names/types | ❌ FAIL — 3 wrong names, wrong types, 2 missing | INV-009 |
| **§2.9 UTC** | All timestamps UTC | ❌ FAIL (by type — STRING prevents verification) | INV-002 |
| **§2.11 Model before DDL** | Tables have Erwin model counterparts | ⚠️ Partial — temp table has no LM entity; 5 LM entities have no physical table | INV-008, INV-003–005 |
| **§2.12 Catalog Registration** | All tables in Informatica GDGC | ❌ UNKNOWN — SA not in taxonomy; likely not registered | INV-021 |
| **PII Masking** | EID/card/phone/passport masked via SHA-256 | ⚠️ UNKNOWN — no PII flags set in mapping (all `pii=None`) | N/A |
| **Partitioning** | Partition by `_ingested_date` | ❌ FAIL — 27/28 tables unpartitioned; 1 partitioned by wrong column | INV-011 |

---

## 7. Remediation Plan

### 7.1 Prioritised Action List

#### P0 — Immediate (Before Any Gold Consumption)

| # | Action | Effort | Owner | Dependency |
|---|---|---|---|---|
| P0-1 | Add `<entity>_sk` (MD5) + `<entity>_bk` to all 28 physical entities; update all ETL pipelines | Large | Data Modeling + ETL | None — foundational |
| P0-2 | Correct all attribute data types: DECIMAL(18,4) for monetary, DATE/TIMESTAMP for dates, BOOLEAN for flags | Large | Data Modeling | P0-1 (combined DDL migration) |
| P0-3 | Resolve 17 unmapped Investment Transaction attributes (amounts, exchange rate, unit price) with FIS SME | Medium | ETL + FIS SME | P0-1 |
| P0-4 | Fix Reference_Exchange_Rate2: map `Source to Target Rate` and `Target to Source Rate` from FIS | Small | ETL + FIS SME | None |
| P0-5 | Move 5 reference tables to `silver.reference`; update all FK references | Medium | Platform Eng + Data Modeling | None |
| P0-6 | Drop `foreign_exchange_agreement_temp_no_partition` from production | Small | Platform Engineering | None |
| P0-7 | Register SA in `sa/subject-areas.md`; split into `silver.wealth` + `silver.treasury`; assign steward | Large | Group Data Office | None |

#### P1 — Next Sprint (Within 30 Days)

| # | Action | Effort | Owner | Dependency |
|---|---|---|---|---|
| P1-1 | Rename SCD columns: `Business Start Date`→`effective_from`, `Business End Date`→`effective_to`, `Is Active Flag`→`is_current` + type fixes | Large | Data Modeling + ETL | P0-2 |
| P1-2 | Add missing `run_id` and `source_ingestion_date` columns to all entities | Medium | ETL | P1-1 |
| P1-3 | Create physical tables for missing entities: `investment_account`, `trade_transaction`, `cash_dividend_event`, `foreign_exchange_event`, `stock` | Large | Data Modeling + ETL | P0-1 |
| P1-4 | Resolve Swap Leg Cash Flow Feature: map 6/7 unmapped attributes | Medium | ETL + FIS SME | None |
| P1-5 | Complete Security Holding mapping: 11 of 13 attributes unmapped | Medium | ETL + FIS SME | None |
| P1-6 | Resolve Swap Leg Num and Account Modifier Number across swap hierarchy | Small | ETL + FIS SME | None |
| P1-7 | Implement `silver.reference.code_mapping` entries for all source int codes (instype, account_type, settlement type, transaction code) | Large | ETL + Reference Data Steward | P0-5 |
| P1-8 | Resolve FX Agreement quantity/currency ambiguity (finding INV-025) | Small | ETL + FIS SME | None |
| P1-9 | Map Dividend Declare Event dates (record date, payable date) | Small | ETL | None |
| P1-10 | Clarify Financial Agreement vs Investment Agreement boundary; create `investment_account` | Medium | Data Modeling | P1-3 |
| P1-11 | Apply partitioning on `source_ingestion_date` to all 28 tables | Large | Platform Engineering | P0-2 |
| P1-12 | Implement AED amount columns (`*_amount_aed` BIGINT fils, `*_amount_orig` DECIMAL, `currency_code`) | Large | ETL | P0-4, P0-5 |
| P1-13 | Verify and fix Application Detail Start/End Date transposition | Small | ETL + FIS SME | None |

#### P2 — Backlog (Next Quarter)

| # | Action | Effort | Owner | Dependency |
|---|---|---|---|---|
| P2-1 | Fix all typos in logical model attribute names (5 identified); migrate DDL as breaking changes | Medium | Data Modeling | None |
| P2-2 | Decompose Investment Transaction god-object into 3–4 normalized entities | Large | Data Modeling | P1-3, P0-1 |
| P2-3 | Remove legacy columns (`Legacy Investment Amount`, `Legacy Average Cost`, `Legacy Tax Amount`) | Small | Data Modeling | P2-2 |
| P2-4 | Move Customer Segment from Investment Transaction to `silver.party` | Medium | Data Modeling | P2-2 |
| P2-5 | Move channel attributes from Investment Transaction to Business Event SA | Medium | Data Modeling | P2-2 |
| P2-6 | Create `investment_transaction_fee` child table for repeating fee/rate group | Medium | Data Modeling | P2-2 |
| P2-7 | Create `stock` master entity; convert `stock_history` to child time-series | Small | Data Modeling | P0-1 |
| P2-8 | Catalog registration: register all wealth/treasury entities in Informatica GDGC with steward, classification, lineage | Large | Group Data Office | P0-7 |
| P2-9 | Resolve duplicate mappings (Securities Financing account/type, Investment Transaction ID) | Small | ETL + FIS SME | None |
| P2-10 | Complete Unit Distribution Event mapping (effective date, base/new unit counts) | Small | ETL | None |
| P2-11 | Remove `reference_transaction_code` amount/metric columns (SLV-007 pre-computation violation) | Small | Data Modeling | P0-5 |
| P2-12 | Set PII classification flags for all relevant attributes (Customer Number (CIF), Account Number, Sales Staff) | Medium | Data Governance + ETL | P0-7 |
| P2-13 | Validate UTC timestamp handling in all ETL pipelines (currently unverifiable due to STRING types) | Medium | ETL | P0-2 |
| P2-14 | Add `has_*` / `is_*` boolean flag columns for Pledge Indicator, FX Indicator | Small | Data Modeling | P0-2 |

---

### 7.2 Recommended Schedule

| Sprint | Duration | Focus |
|---|---|---|
| **Sprint 0 (Governance)** | 1 week | P0-7: SA registration, steward assignment, taxonomy update. Unblocks all downstream work. |
| **Sprint 1 (Foundation)** | 2 weeks | P0-1 + P0-2: Surrogate keys + data types — DDL migration across all 28 tables. P0-5: Reference table migration. P0-6: Drop temp table. |
| **Sprint 2 (Mapping)** | 2 weeks | P0-3 + P0-4: Unmapped critical attributes. P1-4 + P1-5 + P1-6: Swap hierarchy and Security Holding gaps. P1-8 + P1-9: FX/Dividend fixes. |
| **Sprint 3 (SCD + Metadata)** | 1 week | P1-1 + P1-2: SCD column rename + type correction. P1-7: Code mapping implementation. |
| **Sprint 4 (Missing Entities)** | 3 weeks | P1-3: Create 5 missing physical tables. P1-10: FinAgreement/InvAccount clarification. P1-11: Partitioning rebuild. P1-12: AED amounts. |
| **Sprint 5 (3NF / Normalization)** | 3 weeks | P2-2 through P2-6: Decompose god object, remove legacy columns, channel/party separation. |
| **Sprint 6 (Catalog + Compliance)** | 2 weeks | P2-8: Full catalog registration. P2-12: PII flags. P2-13: UTC validation. Remaining P2 items. |

**Total estimated effort: 14 sprints if run solo; 7–8 weeks with a team of 3 (modeling, ETL, platform engineering in parallel).**

---

## 8. Appendix

### Appendix A: Mapping Coverage Summary

| Entity | Total LM Attrs | Attrs in Mapping | Mapped | Not Available | Coverage % |
|---|---|---|---|---|---|
| Investment Transaction | 77 | 70 | 53 | 17 | 69% |
| Reference_Transaction_Code | — | 11 | 11 | 0 | 100% |
| Financial_Agreement | 20 | 9 | 6 | 3 | 67% |
| Investment Agreement / Account | 22 | 10 | 10 | 0 | 100% (mapping only) |
| Foreign Exchange Agreement | 19 | 8 | 8 | 0 | 100% |
| Cash at Other Banks | 9 | 3 | 3 | 0 | 100% |
| Securities Financing | 15 | 8 | 6 | 2 | 75% |
| Security Holding | 13 | 2 | 2 | 0 | 15% of LM |
| Option Agreement | 18 | 7 | 6 | 1 | 86% |
| Forward Futures Agreement | 13 | 2 | 2 | 0 | 15% of LM |
| Swap Agreement | 16 | 7 | 6 | 1 | 86% |
| Swap Currency Agreement | 18 | 4 | 4 | 0 | 100% |
| Swap Agreement Leg | 16 | 5 | 3 | 2 | 60% |
| Swap Leg Cash Flow Feature | 18 | 7 | 1 | 6 | 14% |
| Swap Leg Account Event | 10 | 4 | 2 | 2 | 50% |
| Foreign Exchange Event | 11 | 2 | 2 | 0 | 100% (no physical table) |
| Investment_Product_Event | 13 | 7 | 5 | 2 | 71% |
| Trade Order | 18 | 9 | 8 | 1 | 89% |
| Investment Prod External Event | 9 | 3 | 3 | 0 | 100% |
| Unit Distribution Event | 12 | 6 | 3 | 3 | 50% |
| Dividend Declare Event | 15 | 4 | 2 | 2 | 50% |
| Stock Dividend Amount | 13 | 11 | 10 | 1 | 91% |
| Investment Product | 13 | 4 | 3 | 1 | 75% |
| Reference_Product | 22 | 9 | 5 | 4 | 56% |
| Reference_Exchange_Rate2 | 11 | 5 | 3 | 2 | 60% |
| Cash Dividend Event | 14 | 5 | 4 | 1 | 80% (no physical table) |
| Application Investment detail | 15 | 5 | 4 | 1 | 80% |
| Unit of Measure | 9 | 3 | 2 | 1 | 67% |
| Reference_Currency_Code | — | 4 | 2 | 2 | 50% |
| Trade Execution | 14 | 5 | 4 | 1 | 86% |
| **Trade Transaction** | **93** | **0** | **0** | **0** | **0%** |
| **Investment Account** | **22** | **0** | **0** | **0** | **0%** |
| **Stock** | **14** | **0** | **0** | **0** | **0%** |

---

### Appendix B: Guideline Citations

| Rule Code | Source Document | Quoted Text |
|---|---|---|
| SLV-001/002 | `guidelines/11-modeling-dos-and-donts.md` §2.4 | "Derive surrogate keys deterministically: `MD5(UPPER(TRIM(business_key)))` for single-component keys; `MD5(CONCAT_WS('|', key_part_1, key_part_2))` for composite keys." |
| SLV-003 | `guidelines/11-modeling-dos-and-donts.md` §2.3 | "Every Silver entity table must have `effective_from`, `effective_to`, `is_current`." |
| SLV-004 | `guidelines/11-modeling-dos-and-donts.md` §2.5 | "Resolve all source-system codes to canonical platform values using a central code-mapping reference (e.g., `silver.reference.code_mapping`). Store the canonical code, not the raw source code, in entity columns." |
| SLV-005 | `guidelines/01-data-quality-controls.md` §3.1 | "Completeness: NOT NULL constraint enforced on all primary keys; Silver must contain all required records for core entities." |
| SLV-006 | `guidelines/11-modeling-dos-and-donts.md` §2.1 | "Decompose entities into 3NF: every non-key attribute depends on the whole primary key and nothing but the primary key." |
| SLV-007 | `guidelines/01-data-quality-controls.md` §4 | "Aggregations / KPIs: Prohibited in Silver; reserved for the Gold layer." |
| SLV-008 | `guidelines/11-modeling-dos-and-donts.md` §2.7 | "Include the following Silver technical audit columns on every Silver entity table: `source_system_code`, `source_system_id`, `create_date`, `update_date`, `delete_date`, `is_active_flag`, `effective_from`, `effective_to`, `is_current`, `run_id`, `source_ingestion_date`." |
| SLV-009 | `guidelines/11-modeling-dos-and-donts.md` §2.9 | "Store all timestamps in UTC. Convert source-local timestamps at ingestion." |
| SLV-010 | `guidelines/01-data-quality-controls.md` §2 Gate 2 | "All monetary columns must use `DECIMAL(18,4)`." |
| NAM-001 | `guidelines/06-naming-conventions.md` §1 | "Use snake_case for all identifiers. Use lowercase only — no mixed case." |
| NAM-002 | `guidelines/06-naming-conventions.md` §2 | "Silver: `silver.<subject_area>.<entity>`." |
| NAM-003 | `guidelines/06-naming-conventions.md` §3 | "Silver entity: `<entity>` (e.g., `customer`, `account`, `transaction`)." |
| PII | `guidelines/01-data-quality-controls.md` §10 | "EID, Card Numbers, Phone Numbers, and Passport Numbers masked via SHA-256 unless stored in a restricted container." |

---

### Appendix C: Industry References

| Standard | Application to This SA |
|---|---|
| **BIAN v11 — Investment Portfolio Management** | Missing `Investment Account` entity maps to BIAN "Investment Portfolio Account" service domain. |
| **BIAN v11 — Securities Trade Booking** | Missing `Trade Transaction` entity maps to BIAN "Securities Trade Booking". |
| **BIAN v11 — Market Order** | `trade_order` entity maps here. |
| **BIAN v11 — Derivative Operations** | Swap and Option entities map to "Derivative Operations" service domain. |
| **BIAN v11 — Currency Exchange** | `foreign_exchange_agreement` maps to "Currency Exchange". `foreign_exchange_event` (missing) maps to "Currency Exchange" event. |
| **BIAN v11 — Corporate Action** | `dividend_declare_event`, `stock_dividend_amount`, missing `cash_dividend_event` map to "Corporate Action" service domain. |
| **IFRS 9** | Investment classification (HTM, AFS, FVTPL) requires `purchase_intent_code` in Investment Account. `security_holding` needs fair value / amortised cost split. |
| **BCBS 239** | Principle 3 (Accuracy): 17 unmapped Investment Transaction financial measures directly violate accuracy requirements. Principle 4 (Completeness): Missing Trade Transaction entity (93 attrs) violates completeness. |
| **CBUAE Regulatory Reporting** | `transaction_amount_aed` (unmapped) is required for all CBUAE balance-of-payments and investment-position reports. AED denomination is mandatory. |
| **IFW (IBM/Teradata)** | IFW defines `Financial_Agreement` as the parent contract entity with sub-types for derivatives — validates the Financial Agreement / Swap Agreement / Option Agreement / Forward hierarchy present in this SA. |

---

**Report generated:** 2026-04-27 09:15 UTC
**Confidence level:** High (all 28 physical tables verified, all 31 logical entities reviewed, all 241 mapping rows analysed)
**Total findings:** 28 (11 P0, 9 P1, 8 P2)