# Silver Layer — Executive Gap Assessment Summary
## RAKBANK Data Platform | Group Data Office

| Field | Value |
|---|---|
| **Report Date** | 2026-04-27 |
| **Scope** | 11 Silver Subject Areas |
| **Prepared By** | Automated Gap Assessment (LLM Senior Data Architect) |
| **Purpose** | Executive presentation — design reuse, remediation, and new-build sizing |
| **Source Reports** | `sa/*/output/*_gap_report.md` |

---

## 1. Portfolio Health at a Glance

> **Bottom line:** Every one of the 11 assessed Silver subject areas is rated 🔴 Critical / Not Production-Ready. No subject area is currently safe to promote to the Gold layer or connect to a regulatory reporting pipeline.

| Subject Area | Canonical Schema | Priority | P0 | P1 | P2 | Total Findings | Overall |
|---|---|---|---|---|---|---|---|
| Credit Card | `silver.card` | 2 — High | 12 | 22 | 14 | **48** | 🔴 Critical |
| Customer | `silver.party` ¹ | 1 — Critical | 17 | 14 | 9 | **40** | 🔴 Critical |
| Transaction | `silver.transaction` | 2 — High | 6 | 9 | 5 | **20** | 🔴 Critical |
| Account | `silver.account` | 1 — Critical | 13 | 5 | 2 | **20+** | 🔴 Critical |
| Loan | `silver.loan` | 2 — High | 7 | 9 | 9 | **~25** | 🔴 Critical |
| Deposit | `silver.deposit` | 2 — High | 8 | 11 | 7 | **~26** | 🔴 Critical |
| General Ledger | `silver.gl` | 1 — Critical | 14 | 18 | 9 | **41** | 🔴 Critical |
| Trade | `silver.trade_finance` + `silver.treasury` ² | 4 | 14 | 22 | 18 | **54** | 🔴 Critical |
| Investments | `silver.wealth` + `silver.treasury` ³ | 4 | 11 | 9 | 8 | **28** | 🔴 Critical |
| Collateral | `silver.collateral` | 2 — High | 18 | 19 | 10 | **47** | 🔴 Critical |
| Customer Risk Rating | `silver.risk` (sub-domain) | 3 — Medium | ~10 | ~9 | ~0 | **~19** | 🔴 Critical |
| **TOTAL** | | | **~130** | **~147** | **~91** | **~368** | **All Critical** |

¹ `customer` SA does not exist in the canonical taxonomy — Customer is a Party role. The entire SA must be evaluated for consolidation into `silver.party`.  
² `silver.trade` is not a canonical schema. The physical schema conflates Trade Finance, Treasury/Investments, and Reference Data into one schema; it must be split.  
³ `silver_dev.investments` is not registered in the canonical subject-area taxonomy. It conflates Wealth (client book) and Treasury (bank book) without separation.

---

## 2. Universal Defects — Affecting All 11 Subject Areas

The following structural defects are present in **every** subject area without exception. These are the highest-priority platform-wide remediations regardless of which SA is tackled first.

| # | Defect | Rule Violated | Business Impact | Estimated Fix Effort |
|---|---|---|---|---|
| 1 | **Zero surrogate keys** — No `<entity>_sk` (MD5) or `<entity>_bk` on any entity in any SA | SLV-001 / SLV-002 | Gold-layer FK joins will fail; SCD-2 history cannot be maintained; pipeline re-runs destroy downstream references | Large — must be done per-entity in sequence; affects all pipelines |
| 2 | **All attributes typed STRING / CHAR(18)** — Monetary amounts, dates, timestamps, booleans, identifiers uniformly `STRING` across all SAs (~3,000+ attributes) | SLV-010 / DQ Gate 2 | Financial calculations will silently produce wrong results; IFRS 9 / BCBS 239 computations are unreliable; CBUAE reporting is untrustworthy | Large — requires data type migration + pipeline rewrite for every table |
| 3 | **SCD-2 columns absent or wrong** — `Business Start Date`/`Business End Date`/`Is Active Flag` used instead of `effective_from TIMESTAMP`/`effective_to TIMESTAMP`/`is_current BOOLEAN` | SLV-003 | Point-in-time queries impossible; regulatory time-series reporting broken; Gold models cannot reconstruct history | Large — column rename + type change + SCD-2 pipeline logic across all SAs |
| 4 | **Mandatory metadata columns missing** — `run_id`, `source_ingestion_date`, `delete_date`, and sometimes `create_date`/`update_date` absent from all or most entities | SLV-008 | No audit trail; BCBS 239 data lineage violated; pipeline debugging impossible | Medium — additive change, can be done incrementally |
| 5 | **Wrong catalog / schema names** — All physical schemas use `silver_dev.*` prefix instead of production `silver.*`; several SAs also have wrong schema names vs. canonical taxonomy | NAM-002 | Downstream Gold consumers reading from dev catalog; catalog registration impossible; cross-SA FK references are ambiguous | Medium — schema rename + consumer pipeline update |

---

## 3. Subject Area Reuse / Fix / Build Breakdown

### 3.1 Framework Definitions

| Category | Definition |
|---|---|
| ✅ **Reuse** | Physical table exists, >60% attribute mapping is confirmed, business structure is broadly correct. Can be promoted with tactical fixes (types, keys, SCD columns). Existing ETL pipelines produce usable data today. |
| 🔧 **Fix** | Physical table exists but requires significant remediation before Gold promotion. Data is there; structure must be corrected. Existing pipelines need to be modified, not replaced. |
| 🔨 **Build** | No physical table exists, 0% mapping, entity concept is absent, OR the subject area architecture requires a full decomposition/redesign. New pipelines, new DDL, and new logical model entries required. |

### 3.2 Per-Subject-Area Breakdown

---

#### Credit Card (`silver.card`)

| Dimension | Assessment |
|---|---|
| **Logical Entities** | 50 (7 core, 43 reference) |
| **Physical Tables** | 37 (incl. 3 temp/test tables that must be dropped) |
| **Overall Mapping** | Transaction 96% ✅ · Account 80% 🟡 · Statement 74% 🟡 · Invoice 0% 🔴 |

| Verdict | Entities | Description |
|---|---|---|
| ✅ Reuse | Credit_Card_Transaction | 96% mapped; data is largely correct — needs keys, type fixes, SCD columns only |
| 🔧 Fix | Credit_Card_Account, Credit_Card_Statement, Credit_Card_Statement_Line_Item, Credit_Card_Payment_Schedule, Credit_Card_Receivables | Physical tables exist; 74–80% mapped; require type migration, surrogate key addition, SCD columns, 3NF normalization (repeating arrears groups), and temp table removal |
| 🔨 Build | Credit_Card_Invoice (0% mapped, no table), Card Authorization entity, Card Audit Log entity, 14 misscoped entities to migrate out | Entire billing lifecycle and authorization history is missing |

**Summary:** ~15% Reuse · ~45% Fix · ~40% Build

---

#### Customer (`silver.party`)

| Dimension | Assessment |
|---|---|
| **Logical Entities** | 31 (11 core + 20 reference) |
| **Physical Tables** | 10 (23 of 31 logical entities have no physical table) |
| **Overall Mapping** | 41% (152 of 375 attributes mapped) |

| Verdict | Entities | Description |
|---|---|---|
| ✅ Reuse | None | The SA itself is architecturally non-canonical. No entity can be reused without first resolving the party/customer consolidation governance decision |
| 🔧 Fix | The 10 existing physical tables (`customer`, `customer_contactinformation`, `customer_documents`, `customer_demographics`, `customer_relationships`, `reference_country`, `reference_customer_type`, `reference_segment`, and 2 others) | Tables have data but are incorrectly typed, missing keys, missing SCD columns, and scoped to the wrong schema |
| 🔨 Build | 21 logical entities with no physical table; canonical `party` entity (does not exist); 20 misplaced reference entities need migration to `silver.reference`; entire `silver.party` architecture must be designed | SA consolidation with Party is a governance decision first, then build |

**Summary:** ~5% Reuse · ~30% Fix · ~65% Build/Restructure

---

#### Transaction (`silver.transaction`)

| Dimension | Assessment |
|---|---|
| **Logical Entities** | 18 core + 14 to migrate out |
| **Physical Tables** | 20 (2 separate schemas: electronic + non-electronic) |
| **Overall Mapping** | ~55%; Internet/Mobile 100% ✅ · Branch 35% 🔴 · PhoneBanking 6% 🔴 · Reference_Channel 0% 🔴 |

| Verdict | Entities | Description |
|---|---|---|
| ✅ Reuse | Internet/Mobile Banking Transaction (100%), Payment Platform Transaction (77%), Non-Electronic Transaction – POS (71%) | Highest-mapped entities; data is usable with structural fixes |
| 🔧 Fix | Electronic Channel Transaction (48%), Non-Electronic – ATM (57%), Non-Electronic – Branch (35%), Reference_Merchant, Reference_MCC, Reference_Transaction_Code, Reference_Currency | Physical tables exist; partial mapping; need type/key/SCD fixes + migration of reference tables to `silver.reference` |
| 🔨 Build | Double-entry posting entity (entirely absent — BIAN Financial Transaction pattern), PhoneBanking entity (~6% mapped, no usable data), Reference_Channel/Channel_Type/Channel_Status/ATM (all 0% mapped), Fee Reversal entity, Interest Capitalization entity, schema rename (`silver.transaction`) | Core financial accounting pattern is missing entirely |

**Summary:** ~25% Reuse · ~50% Fix · ~25% Build

---

#### Account (`silver.account`)

| Dimension | Assessment |
|---|---|
| **Logical Entities** | 30 |
| **Physical Tables** | 18 found — but only 2 correspond to logical entities; 15 physical tables have no logical entity counterpart |
| **Overall Mapping** | ~81% on entities that are mapped; but only 2 of 30 logical entities have a physical table |

| Verdict | Entities | Description |
|---|---|---|
| ✅ Reuse | `account_eod_balance` (8.8 GB, has data) | Only physical asset worth carrying forward; needs logical model entry + metadata columns |
| 🔧 Fix | `account_agreement_master`, `account_balance_summary_dd`, `dormant_account` | Data is partially correct; needs decomposition (joint holders, product description, KYC fields to separate SAs), type fixes, key addition, semantic error corrections |
| 🔨 Build | Canonical `account` entity with `account_sk`/`account_bk` (does not exist anywhere); 28 logical entities with no physical table; 3 source-system-prefixed tables (`pbd_`, `ccod_`, `sbca_`) to be unified; 6–9 tables to be relocated to other SAs | The foundation of the SA — the `account` entity — is entirely absent |

**Summary:** ~10% Reuse · ~25% Fix · ~65% Build

---

#### Loan (`silver.loan`)

| Dimension | Assessment |
|---|---|
| **Logical Entities** | 13 + 20 reference |
| **Physical Tables** | 8 active; 5 critical entities have no physical table; 5 tables are empty |
| **Overall Mapping** | Loan_Account 87% ✅ · Loan_Transaction 86% ✅ · Loan_Term_Account 100% ✅ · IFRS9_ECL_Staging 0% 🔴 · Loan_Instalment_Schedule 0% 🔴 · Loan_Statement 0% 🔴 |

| Verdict | Entities | Description |
|---|---|---|
| ✅ Reuse | Loan_Transaction (86%), Loan_Term_Account (100%), Credit_Agreement (92%), Loan_Provision_Change_History (100%) | Good mapping coverage; pipelines producing data; need structural fixes only |
| 🔧 Fix | Loan_Account (87% mapped but 141-attribute god-table mixing 8+ concerns: collection, credit score, disbursement, ESG, MTD aggregates — all must be decomposed) | Most critical asset; has the data but the structure is severely wrong |
| 🔨 Build | `ifrs9_ecl_staging` (entirely absent — P0 regulatory gap), `loan_agreement`, `loan_instalment_schedule`, `loan_statement`, `loan_statement_line_item`, `loan_provision_balance_history` (all missing physical tables), `loan_disbursement` entity, `loan_esg_indicator` entity | IFRS 9 compliance is blocked entirely; instalment schedule and statement lifecycle are absent |

**Summary:** ~30% Reuse · ~40% Fix · ~30% Build

---

#### Deposit (`silver.deposit`)

| Dimension | Assessment |
|---|---|
| **Logical Entities** | ~22 modeled; 26 physical tables found |
| **Physical Tables** | 26 total; ~6 correctly scoped; ~14 belong to other SAs; 3 legacy/temp artifacts to drop |
| **Overall Mapping** | 41% (104/252); `deposit_account` only 29% mapped; `term_deposit_transaction` 76% |

| Verdict | Entities | Description |
|---|---|---|
| ✅ Reuse | `term_deposit_transaction` (76%), `fund_transfer_event` (100%) | Best-mapped in-scope entities |
| 🔧 Fix | `deposit_account` (33.9 GB — largest asset, only 29% mapped; all metadata missing), `deposit_term_account`, `deposit_slip`, `safe_deposit_account`, `deposit_transaction_account` | Core assets with data; need full structural remediation + mapping completion |
| 🔨 Build | 14 out-of-scope tables to migrate (Collateral → `silver.collateral`, GL → `silver.gl`, Payments → `silver.payment`, Reference → `silver.reference`); missing BIAN entities: `deposit_interest_schedule`, `deposit_rollover_instruction`, `deposit_early_redemption`; Islamic finance attributes (profit rate, Sharia structure) | CASA vs. Term Deposit hierarchy needs proper separation; Islamic product support needs to be built |

**Summary:** ~20% Reuse · ~45% Fix · ~35% Build

---

#### General Ledger (`silver.gl`)

| Dimension | Assessment |
|---|---|
| **Logical Entities** | 7 (all have physical tables) |
| **Physical Tables** | 7 |
| **Overall Mapping** | Variable; 3 entities have confirmed wrong-source-column mappings; 2 entities are misscoped |

| Verdict | Entities | Description |
|---|---|---|
| ✅ Reuse | `GL_MASTER` (Chart of Accounts backbone — structure correct in principle), `GL_RECONCILIATION` | Core GL structure is present; needs key/type/SCD fixes |
| 🔧 Fix | `GL_TRANSACTIONS`, `GL_ADJUSTMENTS` (confirmed wrong source-column mapping), `PRODUCT_GL_MAPPING` | Exist and have data; require mapping corrections + structural fixes |
| 🔨 Build | `GL_PERIOD` / Fiscal Calendar (entirely absent — without this, no trial balance or period-end close possible), `GL_BALANCE` / Trial Balance (entirely absent — contradicts stated "Full Balance Sheet and P&L coverage" goal), `GL_ACCOUNT_HIERARCHY` (CoA tree absent), `VAT_RULES` and `CENTRAL_BANK_CODES` to be migrated to `silver.reference` | Core Financial Accounting capabilities are missing; the GL cannot produce Balance Sheet or P&L without these |

**Summary:** ~25% Reuse · ~50% Fix · ~25% Build

---

#### Trade (`silver.trade_finance` + `silver.treasury`)

| Dimension | Assessment |
|---|---|
| **Logical Entities** | 38 (but conflates 4 canonical SAs) |
| **Physical Tables** | 27 (22 with logical entity match); 3 temp/work tables to drop |
| **Overall Mapping** | ~35% fully mapped · ~28% Not Available · ~37% no mapping at all |

| Verdict | Entities | Description |
|---|---|---|
| ✅ Reuse | None — the SA itself must be split before any entity can be declared reusable | Architectural decomposition must happen first |
| 🔧 Fix | After decomposition: `trade_transaction` (but must be split by instrument type), investment/treasury entities (Bond, Fund, FX Agreement, Swap entities), in-scope reference tables | Once the SA split is done, these 22 physical tables can be fixed in place |
| 🔨 Build | `silver.trade_finance` and `silver.treasury` schemas (new canonical schemas), 16 entities with no physical table (Bank Guarantees, Import/Export LC, Trade Counterparty, Invoice Master, Receivable Financing, Trade Exposure Snapshot), per-instrument entity model (Import LC, Export LC, BG as separate entities instead of one super-table), Customer entity to migrate to `silver.party`, 7 reference entities to migrate to `silver.reference` | The canonical Trade Finance domain is almost entirely unbuilt |

**Summary:** ~10% Reuse · ~30% Fix · ~60% Build/Restructure

---

#### Investments (`silver.wealth` + `silver.treasury`)

| Dimension | Assessment |
|---|---|
| **Logical Entities** | 31 (not in canonical taxonomy) |
| **Physical Tables** | 25 (5 missing); 1 temp table to drop |
| **Overall Mapping** | ~156 mapped · ~70 Not Available |

| Verdict | Entities | Description |
|---|---|---|
| ✅ Reuse | After SA split: Swap entities, `trade_order`, `trade_execution`, `security_holding`, `foreign_exchange_agreement`, option/forward agreements — all have physical tables with reasonable data | Bank-book treasury instruments have the most mature coverage |
| 🔧 Fix | After SA split: All swap/FX/option tables need type/key/SCD fixes; `investment_product`, `securities_financing`, dividend events need type corrections; 5 reference tables need migration to `silver.reference` | Once SA is registered and split, these can be fixed incrementally |
| 🔨 Build | `silver.wealth` and `silver.treasury` schema registration and SA taxonomy entries, `Investment_Account` (missing physical), `Trade_Transaction` (missing), `Cash_Dividend_Event` (missing), `Foreign_Exchange_Event` (missing), `Stock` entity (missing), resolution of 17 unmapped critical financial attributes in Investment Transaction | Client-facing wealth management view and several bank-book positions are absent |

**Summary:** ~25% Reuse · ~45% Fix · ~30% Build

---

#### Collateral (`silver.collateral`)

| Dimension | Assessment |
|---|---|
| **Logical Entities** | 12 |
| **Physical Tables** | 10 (2 missing); 7 non-production artifacts + 2 misscoped tables to remove |
| **Overall Mapping** | ~44% (54/122) |

| Verdict | Entities | Description |
|---|---|---|
| ✅ Reuse | `Customer_CollateralAsset` (parent asset record — most complete), `Customer_Collateral_Real_Estate`, `Customer_Collateral_Vehicles` | Best-mapped entities; data exists and is broadly correct domain-wise |
| 🔧 Fix | All 10 existing physical tables: remove 7 non-production artifacts, remove 2 misscoped transaction tables, fix all CHAR(18) types, add surrogate keys, standardize SCD columns, complete mapping for 26 unmapped attributes across sub-asset entities | All existing tables need structural remediation |
| 🔨 Build | `collateral_valuation_history` (absent — LTV tracking impossible), `collateral_lien_registration` (absent — IFRS 9 collateral-coverage calculation blocked), `Customer_Collateral_IP` and `Customer_Collateral_Inventory` (no physical tables) | IFRS 9 LGD collateral-coverage engine cannot function without valuation history |

**Summary:** ~20% Reuse · ~55% Fix · ~25% Build

---

#### Customer Risk Rating (`silver.risk` sub-domain)

| Dimension | Assessment |
|---|---|
| **Logical Entities** | 8 (7 with physical tables, 2 tables empty) |
| **Physical Tables** | 39 total — but 28 are Bronze-pattern source replicas; only 7 are true Silver entities; 2 are prohibited pre-aggregated tables |
| **Overall Mapping** | 34 of 93 mapped attributes have issues; no data-type specification in any mapping row |

| Verdict | Entities | Description |
|---|---|---|
| ✅ Reuse | `Customer_RiskRating_Scoring`, `Customer_BureauScore`, `Customer_ScoreHistory` | The core CRR entities have data; structure is broadly correct for the domain |
| 🔧 Fix | `Customer_CollectionSummary`, `Customer_DeliquencyHistory`, `Reference_Risk_Model`, `Reference_Bureau_Source` — all 7 core Silver entities need type/key/SCD/metadata fixes; governance decision needed on Collection overlap with Lending SA | Core entities exist but need structural remediation |
| 🔨 Build | Drop `reference_aggregated_pipelines` and `reference_aggregated_individualcontracts` (prohibited aggregations); remove or reclassify 28 Bronze-pattern staging tables (`reference_custinfo`, `reference_score`, etc.) and replace with proper 3NF Silver entities; remove `Loan_Account` (141 attributes belonging to `silver.lending`); align schema name from `silver_dev.customer_riskrating_scoring` to `silver.risk` | The bulk of the physical footprint (28/39 tables) is Bronze staging in disguise — it must be redesigned as proper Silver |

**Summary:** ~15% Reuse · ~35% Fix · ~50% Build/Restructure

---

### 3.3 Consolidated Portfolio Reuse / Fix / Build

> **How to read this table:** Percentages are effort/entity-weighted estimates based on entity count, physical table existence, mapping completeness, and structural correctness assessed in each individual gap report.

| Subject Area | ✅ Reuse | 🔧 Fix | 🔨 Build | Confidence |
|---|---|---|---|---|
| Credit Card | 15% | 45% | 40% | High |
| Customer | 5% | 30% | 65% | High |
| Transaction | 25% | 50% | 25% | High |
| Account | 10% | 25% | 65% | High |
| Loan | 30% | 40% | 30% | High |
| Deposit | 20% | 45% | 35% | High |
| General Ledger | 25% | 50% | 25% | High |
| Trade | 10% | 30% | 60% | High |
| Investments | 25% | 45% | 30% | High |
| Collateral | 20% | 55% | 25% | Medium |
| Customer Risk Rating | 15% | 35% | 50% | High |
| **Portfolio Average** | **~18%** | **~41%** | **~41%** | |

#### Visual Summary

```
Portfolio-Level Design Disposition (by weighted effort)

  ✅ REUSE   ██████░░░░░░░░░░░░░░░░░░░░░░░░░░  ~18%
  🔧 FIX     ████████████████░░░░░░░░░░░░░░░░  ~41%
  🔨 BUILD   ████████████████░░░░░░░░░░░░░░░░  ~41%
```

**Key takeaway for the presentation:**
- Only **~18% of the current design can be carried forward as-is** (with tactical structural fixes).
- **~41% needs significant remediation** — the data and pipelines exist but the structure must be corrected across all 5 universal defects before Gold promotion.
- **~41% must be built from scratch** — either because physical tables are absent, mapping is 0%, or the subject area architecture must be fully redesigned (Customer → Party, Trade → Trade Finance + Treasury, Investments → Wealth + Treasury).

---

## 4. Architecture Restructuring Required

Three subject areas require more than a fix — they require a fundamental architectural restructuring:

| SA | Current State | Required Target |
|---|---|---|
| **Customer** | `silver_dev_v2.customer` — SA not in canonical taxonomy | Consolidate into `silver.party`; governance decision required on customer-as-Party-role implementation |
| **Trade** | `silver_dev.trade` — conflates Trade Finance + Treasury + Reference + Party | Split into `silver.trade_finance` (LC, BG, Bills of Exchange, Incoterms) and `silver.treasury` (Bonds, Sukuk, Derivatives, FX, Money Market) |
| **Investments** | `silver_dev.investments` — not in taxonomy; conflates client Wealth and bank Treasury | Register `silver.wealth` (client investment portfolios) and expand `silver.treasury` (bank-book positions) |

These restructurings affect cross-SA FK dependencies and must be governed by the Group Data Office before pipelines are re-wired.

---

## 5. Critical Entities Missing Across the Portfolio

The following entities are **entirely absent** (no physical table, no mapping) and directly block regulatory or analytical use-cases:

| Missing Entity | Subject Area | Blocking Use-Case | Severity |
|---|---|---|---|
| `ifrs9_ecl_staging` | Loan | IFRS 9 Expected Credit Loss staging (Stage 1/2/3) — entire ECL engine blocked | 🔴 P0 |
| `GL_BALANCE` / Trial Balance | GL | Balance Sheet and P&L production impossible | 🔴 P0 |
| `GL_PERIOD` / Fiscal Calendar | GL | Period-end financial close impossible | 🔴 P0 |
| `collateral_valuation_history` | Collateral | IFRS 9 collateral LGD coverage calculation blocked | 🔴 P0 |
| `credit_card_invoice` | Credit Card | Entire revolving credit billing lifecycle missing | 🔴 P0 |
| Double-entry posting entity | Transaction | Atomic financial ledger posting — foundational for any P&L reconciliation | 🔴 P0 |
| `collateral_lien_registration` | Collateral | IFRS 9 collateral-coverage and lien priority tracking blocked | 🔴 P0 |
| `loan_agreement` | Loan | Legal agreement lifecycle for credit facilities absent | 🔴 P0 |
| `GL_ACCOUNT_HIERARCHY` | GL | Chart of Accounts tree / multi-level reporting impossible | 🔴 P1 |
| `loan_instalment_schedule` | Loan | Repayment schedule, DPD calculation, BCBS 239 EAD impossible | 🔴 P0 |
| Card Authorization entity | Credit Card | Authorization + settlement reconciliation impossible | 🔴 P1 |
| `party` canonical entity | Customer | Single source of truth for customer identity absent | 🔴 P0 |

---

## 6. Common Root Causes

| Root Cause | SAs Affected | Management Action Required |
|---|---|---|
| **Logical model predates platform standards** — Erwin model was built before Silver guidelines were issued; SCD, SK, and type conventions were not applied | All 11 | Retrospective model alignment sprint; update Erwin templates with mandatory column standards |
| **Dev catalog in production use** — `silver_dev.*` schemas contain production data but are named as development catalogs | All 11 | Catalog promotion plan; rename or migrate to `silver.*` catalog before any Gold build |
| **All-STRING antipattern** — Source system attributes copied verbatim without type canonicalization | All 11 | Enforce ETL type-casting at Bronze→Silver promotion; gate pipeline promotion on DQ Gate 2 |
| **Missing surrogate key standard** — No MD5 SK pattern was enforced when tables were first deployed | All 11 | Issue SLV-001/002 waiver for existing tables and mandate backfill; enforce in all new pipelines |
| **Scope contamination** — Reference tables, payment events, GL entries, collateral items, etc. deployed into wrong SA schemas | 9 of 11 | Cross-SA data migration plan; Reference Data SA must be the first SA brought to production |
| **Reference Data SA not yet built** — `silver.reference` is referenced as the canonical destination for 60+ misplaced reference tables but is not a built-out SA | All SAs | Immediate: bootstrap `silver.reference` schema and migrate ISO Currency, Country, Exchange Rate, Transaction Codes as a prerequisite for all other SA remediations |

---

## 7. Prioritized Remediation Roadmap

### Phase 0 — Platform Prerequisites (Weeks 1–4, all SAs)

These actions are prerequisites for all per-SA remediations. They unblock every other fix.

| # | Action | Owner | Effort |
|---|---|---|---|
| 0-1 | Bootstrap `silver.reference` SA — create schema, migrate ISO Currency, Country, Exchange Rate, Transaction Code tables from all polluted SAs | Reference Data Owner + DBA | Medium |
| 0-2 | Issue unified Silver metadata standard — resolve naming conflict between `guidelines/01-data-quality-controls.md` and `guidelines/11-modeling-dos-and-donts.md` (5 column name discrepancies); publish single canonical standard | Group Data Office | Small |
| 0-3 | Govern Customer → Party restructuring decision — the entire Customer SA cannot be remediated until governance decides whether it consolidates into `silver.party` | Chief Data Officer + Business | Small |
| 0-4 | Govern Trade + Investments SA split decision — create `silver.trade_finance`, `silver.treasury`, `silver.wealth` schemas in the canonical taxonomy | Group Data Office + Architecture | Small |
| 0-5 | Create Erwin logical model templates with all mandatory columns pre-populated (11 SCD/audit columns, `_sk`/`_bk` placeholders, correct data types) | Data Modeling | Small |

### Phase 1 — P0 Structural Fixes (Sprints 1–3)

Execute per-SA in this sequence (highest regulatory impact first):

| Sprint | Subject Area | Key P0 Deliverables |
|---|---|---|
| Sprint 1 | **GL** | Add SK/BK; fix 3 wrong-source-column mappings; create `GL_PERIOD` and `GL_BALANCE` entities |
| Sprint 1 | **Loan** | Add SK/BK to all tables; create `ifrs9_ecl_staging`; fix all data types |
| Sprint 2 | **Account** | Create canonical `account` entity with SK/BK; remove misscoped tables; fix semantic errors |
| Sprint 2 | **Collateral** | Add SK/BK; create `collateral_valuation_history`; remove 9 non-production artifacts |
| Sprint 3 | **Transaction** | Schema rename to `silver.transaction`; add SK/BK; fix types; remove 14 misscoped entities |
| Sprint 3 | **Deposit** | Add SK/BK; relocate 14 out-of-scope tables; fix types; critical semantic mapping errors |

### Phase 2 — P0 Structural Fixes Continued (Sprints 4–6)

| Sprint | Subject Area | Key P0 Deliverables |
|---|---|---|
| Sprint 4 | **Credit Card** | Add SK/BK; fix all types; build `Credit_Card_Invoice` entity; drop test tables |
| Sprint 4 | **CRR** | Remove Loan_Account; drop prohibited aggregation tables; reclassify 28 Bronze replicas |
| Sprint 5 | **Trade** | SA split into `silver.trade_finance` + `silver.treasury`; add SK/BK to 22 existing tables |
| Sprint 5 | **Investments** | SA registration; split into Wealth/Treasury; add SK/BK; fix types |
| Sprint 6 | **Customer** | Post-governance: begin migration to `silver.party` or stand up `silver.party` alongside |

### Phase 3 — P1 Quality and Completeness (Sprints 7–10)

- SCD-2 column standardization across all SAs (rename from `Business Start Date` → `effective_from`, etc.)
- Migrate all remaining reference tables to `silver.reference`
- Implement DQ gate + quarantine table pattern across all SAs
- Code mapping via `silver.reference.code_mapping` for all raw source codes
- Catalog registration (Unity Catalog / Informatica GDGC) for all Silver tables
- PII masking (SHA-256) for card numbers, EID, phone, passport across Transaction, Credit Card, Customer SAs

---

## 8. Summary for Presentation

### Slide-Ready Bullets

**Current State:**
- 11 Silver subject areas assessed; **all 11 rated Critical / Not Production-Ready**
- **~368 total findings**: ~130 P0 (blocking), ~147 P1 (next sprint), ~91 P2 (backlog)
- **0 subject areas** are safe to connect to Gold layer or regulatory reporting today
- **5 universal defects** affect every table in the platform simultaneously

**Design Reuse vs. Fix vs. Build:**

| | % of Effort | What It Means |
|---|---|---|
| ✅ **Reuse** | ~18% | Data is there, structure is broadly usable — add keys, fix types, add SCD columns |
| 🔧 **Fix** | ~41% | Pipelines and data exist but structure must be corrected before Gold promotion |
| 🔨 **Build** | ~41% | Missing entities, 0% mapping, or SA architecture must be redesigned from scratch |

**Highest-Risk Gaps (Regulatory):**
- **IFRS 9**: ECL staging table missing in Loan; collateral valuation history missing in Collateral; Gold-layer IFRS 9 engine cannot be built
- **BCBS 239**: All-STRING types mean accuracy and completeness principles are violated; no data lineage (missing `run_id`, `source_ingestion_date`) across all SAs
- **CBUAE Reporting**: GL Trial Balance impossible (GL_BALANCE/GL_PERIOD missing); Transaction schema name divergence means CBUAE reports read from wrong catalog
- **AML / KYC**: Customer SA not in canonical taxonomy; CRR has Bronze replicas in Silver; no unified party identity layer

**Recommended Decision:**
> Approve a **4-phase remediation program** (~10 sprints). The first 4 weeks are platform prerequisites (Reference Data SA bootstrap, governance decisions, metadata standard). Sprints 1–6 address all P0 blockers SA by SA. No subject area should be promoted to Gold until its P0 findings are resolved and its mandatory structural columns are in place.

---

*Report generated: 2026-04-27 | Source: automated analysis of 11 individual gap reports under `sa/*/output/` | Distribution: Group Data Office, Data Platform Architecture, Business Sponsors*
