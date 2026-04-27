# Transaction Subject Area — Silver Layer Gap Assessment Report

**Assessment Date:** 2026-04-27  
**Assessor:** Senior Data Platform Architect (automated gap analysis)  
**Subject Area:** Transaction (`silver.transaction`)  
**Priority:** 2 — High  
**Report Version:** 1.0  
**Normative References:** guidelines/01-data-quality-controls.md · guidelines/06-naming-conventions.md · guidelines/11-modeling-dos-and-donts.md

---

## 1. Executive Summary

### 1.1 Management Slide

| Dimension | Finding |
|---|---|
| **Subject Area** | Transaction — The Financial Ledger: Atomic Double-Entry Postings, Card Settlements, POS Transactions, Fee Reversals, and Interest Capitalization |
| **Canonical Schema** | `silver.transaction` (per `sa/subject-areas.md`) |
| **Physical Schemas Found** | `silver_dev.electronic_channel_transaction` and `silver_dev.non_electronic_channel_transaction` — **both deviate from the canonical schema name** |
| **Logical Model Sub-Areas** | `Txn - Electronic Channel Txn` (47 entities, 792 attrs) · `Txn - NonElectronic Channel Txn` (10 entities, 193 attrs) |
| **Physical Tables Found** | 12 tables in electronic schema · 8 tables in non-electronic schema |
| **Total Logical Entities Assessed** | 18 core entities (transaction + reference) + 14 misscoped entities identified for migration |
| **Overall Mapping Completeness** | ~55% across all entities; Phone Banking: ~6%; Reference_ATM: 0% |
| **Critical Gaps (P0)** | 6 — schema mismatch, missing surrogate keys, all-STRING types, misscoped entities, missing double-entry model, Phone Banking unmapped |
| **High Gaps (P1)** | 9 — reference data co-location, empty tables, SCD naming, PII card masking, raw source codes, monetary type, UTC enforcement, duplicate reference tables, missing physical implementations |
| **Medium Gaps (P2)** | 5 — table naming, grain ambiguity, 3NF violations, duplicate logical entity, payment_platform Delta version |
| **Confidence** | 0.92 (high; column-level detail limited to mapping workbook; no DDL available) |

### 1.2 Top 5 Actions (Management Priority)

| # | Action | Priority | Impact | Effort |
|---|---|---|---|---|
| 1 | **Rename schemas to `silver.transaction`** and consolidate electronic + non-electronic into a single subject-area schema | P0 | Architectural alignment; all downstream Gold reads broken otherwise | Medium |
| 2 | **Add `_sk` (MD5 surrogate) and `_bk` columns** to every transactional and reference entity | P0 | SCD-2 history, Gold FK integrity, idempotent pipelines all depend on this | Large |
| 3 | **Fix all data types**: monetary amounts → `DECIMAL(18,4)`, dates → `DATE`, timestamps → `TIMESTAMP` UTC, booleans → `BOOLEAN` | P0 | Type safety; regulatory reporting accuracy; BCBS 239 lineage | Large |
| 4 | **Remove 14 misscoped entities** from `Txn - Electronic Channel Txn` SA (Customer, Event, Account Event, etc.) and migrate to correct SAs (Party, Business Event) | P0 | SA isolation, governance boundary enforcement | Medium |
| 5 | **Migrate all `reference_*` tables to `silver.reference`** and eliminate cross-schema duplication of `reference_bank_branch`, `reference_mcc`, `reference_merchant` | P1 | Single source of truth for reference data; eliminates 3 duplicate tables | Medium |

---

## 2. Subject Area Architecture Assessment

### 2.1 Scoping vs. `sa/subject-areas.md`

**Finding:** The canonical taxonomy defines a single SA **Transaction** with schema `silver.transaction`. The physical implementation uses **two separate schemas**: `silver_dev.electronic_channel_transaction` and `silver_dev.non_electronic_channel_transaction`. The `_dev` catalog suffix indicates these are development-environment objects that may never have been promoted under the correct catalog.

The logical model compounds this by splitting the SA into two separate sub-areas (`Txn - Electronic Channel Txn` and `Txn - NonElectronic Channel Txn`) rather than treating them as implementation partitions of a single unified subject area.

**Verdict:** ❌ **Non-compliant.** The schema names violate NAM-002 (pattern must be `silver.<subject_area>.<entity>`). The `_dev` catalog is non-production and the canonical catalog should be `silver`.

### 2.2 Domain Decomposition vs. BIAN / IFW

The transaction SA description calls for "Atomic Double-Entry Postings, Card Settlements, POS Transactions, Fee Reversals, and Interest Capitalization." Assessment:

| Expected Domain Pattern | Present? | Evidence |
|---|---|---|
| **Atomic Double-Entry Posting** (debit/credit legs, GL posting key) | ❌ Absent | No `transaction_posting` or `financial_ledger_entry` entity exists in either SA or physical schema. The `Electronic Channel Transaction` entity is a channel-level event record, not an atomic posting. |
| **Card Settlement** | ⚠️ Partial | `electronic_channel_transaction` and `internet_mobile_banking_transaction` cover card-channel transactions but mix authorisation + settlement in one entity |
| **POS Transaction** | ✅ Present | `non_electronic_channel_transaction__pos_` (naming issue noted) |
| **ATM Transaction** | ✅ Present | `non_electronic_channel_transaction_atm` |
| **Branch Transaction** | ⚠️ Partial | `non_electronic_channel_transaction__branch_` exists; mapping gaps remain |
| **Fee / Reversal** | ❌ Absent | No distinct fee reversal entity; `reference_transaction_code` has `Fee applicable` and `Reversal allowed` flags but no separate entity |
| **Interest Capitalization** | ❌ Absent | No capitalization posting entity |
| **Payment Platform** | ✅ Present | `payment_platform_transaction` (2.9 GB) |
| **Phone Banking** | ❌ Critically incomplete | `reference_phonebankingagent` is empty; Non Electronic Channel Transaction (PhoneBanking) is ~6% mapped |
| **Channel Reference master** | ⚠️ Partial | `reference_channel`, `reference_channel_type`, `reference_channel_status`, `reference_channel_category_type` present but two are empty; belong in `silver.reference` |

**Verdict:** ❌ **Partial. Core double-entry, fee reversal, and interest capitalization patterns missing.**

### 2.3 Identity Strategy

Checked against **SLV-001** ("`<entity>_sk` MD5 surrogate on every entity") and **SLV-002** ("`<entity>_bk` business key on every entity"):

| Entity | `_sk` Present | `_bk` Present | Compliant |
|---|---|---|---|
| Electronic Channel Transaction | ❌ | ❌ | ❌ |
| Internet/Mobile Banking Transaction | ❌ | ❌ | ❌ |
| Payment Platform Transaction | ❌ | ❌ | ❌ |
| Non Electronic Channel Transaction | ❌ | ❌ | ❌ |
| Non Electronic Channel Transaction (ATM) | ❌ | ❌ | ❌ |
| Non Electronic Channel Transaction (Branch) | ❌ | ❌ | ❌ |
| Non Electronic Channel Transaction (POS) | ❌ | ❌ | ❌ |
| Non Electronic Channel Transaction (PhoneBanking) | ❌ | ❌ | ❌ |
| All Reference entities (10) | ❌ | ❌ | ❌ |

**Verdict:** ❌ **0% compliant. No entity in the Transaction SA carries a surrogate key or tagged business key.**

### 2.4 SCD Strategy

The logical model introduces history-tracking columns but under non-standard names:

| Required Column (guideline) | Logical Model Column | Type in Model | Compliant |
|---|---|---|---|
| `effective_from` TIMESTAMP | `Business Start Date` CHAR(18) | Wrong name + wrong type | ❌ |
| `effective_to` TIMESTAMP | `Business End Date` CHAR(18) | Wrong name + wrong type | ❌ |
| `is_current` BOOLEAN | `Is Active Flag` CHAR(18) | Wrong name + wrong type | ❌ |
| `delete_date` TIMESTAMP | `IsLogicallyDeleted` STRING | Wrong name + wrong type (should be timestamp) | ❌ |
| `run_id` STRING | `Source System Id` (repurposed?) | Missing | ❌ |

**Additional issue:** The `Non Electronic Channel Transaction (POS)` entity has **triplicate** copies of `Business Start Date`, `Business End Date`, `Is Active Flag` with suffixed numeric IDs (`__9057`, `__9058`, etc.) — clear Erwin modeling artefact (copy-paste error) that was not corrected.

**Verdict:** ❌ **SCD-2 columns present conceptually but all use wrong names and wrong data types.**

### 2.5 Cross-Entity Referential Integrity

| FK Relationship | Documented? | FK Column Present | Verdict |
|---|---|---|---|
| `electronic_channel_transaction.channel_bk` → `reference_channel` | ❌ | ❌ (raw `Channel Bank` string) | ❌ |
| `electronic_channel_transaction` → `reference_currency` | ❌ | Via `channel_currency_code` STRING (no FK declared) | ⚠️ |
| `payment_platform_transaction` → `reference_transaction_code` | ❌ | None | ❌ |
| `non_electronic_channel_transaction_atm` → `reference_atm` (missing physical) | ❌ | None | ❌ |
| `non_electronic_channel_transaction__pos_` → `reference_merchant` | ❌ | Via `merchant_id` string | ⚠️ |
| `internet_mobile_banking_transaction` → `reference_channel` | ❌ | None | ❌ |
| `non_electronic_channel_transaction__branch_` → `reference_bank_branch` | ❌ | Via `branch_code` | ⚠️ |
| Transaction entities → `silver.party.customer` | ❌ | Via raw `cif_number`/`customer_number` strings | ⚠️ |
| Transaction entities → `silver.account.account` | ❌ | Via raw `account_number` strings | ⚠️ |

**Verdict:** ❌ **No FK constraints declared. Cross-SA references use raw string keys without FK documentation.**

### 2.6 ETL Mapping Completeness

| Entity | Total Attrs | Mapped | Not Available | % Complete |
|---|---|---|---|---|
| Electronic Channel Transaction | 50 | 24 | 26 | **48%** |
| Internet/Mobile Banking Transaction | 44 | 44 | 0 | **100%** |
| Payment Platform Transaction | 44 | 34 | 10 | **77%** |
| Non Electronic Channel Transaction | 23 | 18 | 5 | **78%** |
| Non Electronic Channel Transaction (Branch) | 17 | 6 | 11 | **35%** |
| Non Electronic Channel Transaction (ATM) | 14 | 8 | 6 | **57%** |
| Non Electronic Channel Transaction (POS) | 14 | 10 | 4 | **71%** |
| Non Electronic Channel Transaction (PhoneBanking) | 17 | 1 | 16 | **6%** |
| Reference_Bank_Branch | 18 | 10 | 8 | **56%** |
| Reference_Merchant | 28 | 21 | 7 | **75%** |
| Reference_MCC | 13 | 9 | 4 | **69%** |
| Reference_Transaction_Code | 24 | 20 | 4 | **83%** |
| Reference_Currency | 11 | 9 | 2 | **82%** |
| Reference_Channel | 14 | 0 | 14 | **0%** |
| Reference_Channel_Type | 10 | 0 | 10 | **0%** |
| Reference_Channel_Status | 8 | 0 | 8 | **0%** |
| Reference_Channel_Category_Type | 9 | 0 | 9 | **0%** |
| Reference_ATM | 14 | 0 | 14 | **0%** |
| Reference_PhoneBankingAgent | 11 | 3 | 8 | **27%** |

**Verdict:** ❌ **Overall mapping completeness ~57%. Phone Banking and all channel reference entities critically behind.**

---

## 3. Entity Inventory

| # | Entity | SA Sub-Area | Physical Table | Status | SK | BK | SCD | Mapping % | Scoping |
|---|---|---|---|---|---|---|---|---|---|
| 1 | Electronic Channel Transaction | Txn - Elect | `electronic_channel_transaction` | ✅ Physical | ❌ | ❌ | ❌ Wrong | 48% | ✅ In-scope |
| 2 | Internet/Mobile Banking Transaction | Txn - Elect | `internet_mobile_banking_transaction` | ✅ Physical | ❌ | ❌ | ❌ Wrong | 100% | ✅ In-scope |
| 3 | Payment Platform Transaction | Txn - Elect | `payment_platform_transaction` | ✅ Physical | ❌ | ❌ | ❌ Wrong | 77% | ✅ In-scope |
| 4 | Non Electronic Channel Transaction | Txn - Non-Elect | `non_electronic_channel_transaction` | ✅ Physical | ❌ | ❌ | ❌ Wrong | 78% | ✅ In-scope |
| 5 | Non Electronic Channel Transaction (ATM) | Txn - Non-Elect | `non_electronic_channel_transaction_atm` | ✅ Physical | ❌ | ❌ | ❌ Wrong | 57% | ✅ In-scope |
| 6 | Non Electronic Channel Transaction (Branch) | Txn - Non-Elect | `non_electronic_channel_transaction__branch_` | ✅ Physical (naming ❌) | ❌ | ❌ | ❌ Wrong | 35% | ✅ In-scope |
| 7 | Non Electronic Channel Transaction (POS) | Txn - Non-Elect | `non_electronic_channel_transaction__pos_` | ✅ Physical (naming ❌) | ❌ | ❌ | ❌ Wrong | 71% | ✅ In-scope |
| 8 | Non Electronic Channel Transaction (PhoneBanking) | Txn - Non-Elect | ⚠️ Served by parent table | ⚠️ Partial | ❌ | ❌ | ❌ Wrong | 6% | ✅ In-scope |
| 9 | Reference_Bank_Branch | Both | `reference_bank_branch` (DUPLICATED) | ✅ but ❌ SA | ❌ | ❌ | ❌ | 56% | ❌ Belongs in `silver.reference` |
| 10 | Reference_MCC | Both | `reference_mcc` (DUPLICATED; electronic = 0B) | ⚠️ Partial | ❌ | ❌ | ❌ | 69% | ❌ Belongs in `silver.reference` |
| 11 | Reference_Merchant | Both | `reference_merchant` (DUPLICATED) | ✅ but ❌ SA | ❌ | ❌ | ❌ | 75% | ❌ Belongs in `silver.reference` |
| 12 | Reference_Transaction_Code | Txn - Elect | `reference_transaction_code` | ✅ Physical | ❌ | ❌ | ❌ | 83% | ❌ Belongs in `silver.reference` |
| 13 | Reference_Currency | Txn - Elect | `reference_currency` | ✅ Physical | ❌ | ❌ | ❌ | 82% | ❌ Belongs in `silver.reference` |
| 14 | Reference_Channel | Txn - Elect | `reference_channel` | ✅ Physical | ❌ | ❌ | ❌ | 0% | ❌ Belongs in `silver.reference` |
| 15 | Reference_Channel_Type | Txn - Elect | `reference_channel_type` (EMPTY) | ❌ Empty | ❌ | ❌ | ❌ | 0% | ❌ Belongs in `silver.reference` |
| 16 | Reference_Channel_Status | Txn - Elect | `reference_channel_status` | ✅ Physical | ❌ | ❌ | ❌ | 0% | ❌ Belongs in `silver.reference` |
| 17 | Reference_Channel_Category_Type | Txn - Elect | `reference_channel_category_type` | ✅ Physical | ❌ | ❌ | ❌ | 0% | ❌ Belongs in `silver.reference` |
| 18 | Reference_ATM | Txn - Non-Elect | ❌ No physical table | ❌ Missing | ❌ | ❌ | ❌ | 0% | ❌ Belongs in `silver.reference` or `silver.org` |
| 19 | Reference_PhoneBankingAgent | Txn - Non-Elect | `reference_phonebankingagent` (EMPTY) | ❌ Empty | ❌ | ❌ | ❌ | 27% | ❌ Belongs in `silver.org` |
| **Misscoped** | Customer (145 attrs) | Txn - Elect | None | ❌ Misscoped | N/A | N/A | N/A | N/A | ❌ Party SA |
| **Misscoped** | Event, Event_Relationship, Event_To_Locator | Txn - Elect | None | ❌ Misscoped | N/A | N/A | N/A | N/A | ❌ Business Event SA |
| **Misscoped** | Account Event, Account_Event1, Account_Group_Event, Financial_Account_Event | Txn - Elect | None | ❌ Misscoped | N/A | N/A | N/A | N/A | ❌ Account SA |
| **Misscoped** | Financial_Event, Fund_Transfer_Event, Interbank_Event, Intrabank_Event, Check_Event | Txn - Elect | None | ❌ Misscoped | N/A | N/A | N/A | N/A | ❌ Payments/Account SA |
| **Misscoped** | Reference_Business_SIC_History | Txn - Elect | None | ❌ Misscoped | N/A | N/A | N/A | N/A | ❌ Reference SA |

---

## 4. Per-Entity Assessments

---

### 4.1 Electronic Channel Transaction

**Physical Table:** `silver_dev.electronic_channel_transaction.electronic_channel_transaction` (14.6 MB)  
**Logical Model:** `Txn - Electronic Channel Txn → Electronic Channel Transaction` (53 attributes)

#### 4.1a Industry Fitness

The entity attempts to capture a channel-level interaction event from PRIME (card/e-channel transactions). Its grain is ambiguous — it contains both the channel event metadata (terminal, response codes, user ID) and the financial amount (Channel Transaction Amount, Currency Code, Service Charge). This mixes two concerns:
- **Channel Event** (who initiated, from where, via which terminal)
- **Financial Movement** (debit/credit amount, account, currency)

Per BIAN Service Domain _Financial Transaction_: the canonical pattern separates the channel interaction from the atomic financial posting. The Transaction SA is specifically for "Atomic Double-Entry Postings" — this entity is a **channel interaction record**, not an atomic ledger posting.

**Verdict:** ⚠️ Partial — entity grain is channel-level, not posting-level. Missing debit/credit indicator; `Channel Bank` and `Channel Branch` are denormalized strings.

#### 4.1b Attribute-Level Review

| Attribute | Logical Type | Mapping Status | Source | Issue |
|---|---|---|---|---|
| Transaction Id | STRING | Mapped | PRIME.CISOTRXNS.I062V2_TRANS_ID | ❌ No `electronic_channel_transaction_sk` (MD5) or `electronic_channel_transaction_bk` |
| Transaction type | STRING | Mapped | PRIME.CTRANSACTIONS.TRXNTYPE | ❌ Raw source code; must use `silver.reference.code_mapping` (SLV-004) |
| Transaction code | STRING | Mapped | PRIME.CTRANSACTIONS.SERNO | ⚠️ Named "Transaction code" but mapped to SERNO (serial number) — semantic mismatch |
| Channel Transaction Date | STRING | Mapped | PRIME.CTRANSACTIONS.I013_TRXN_DATE | ❌ Should be DATE type, not STRING |
| Channel Transaction Source | STRING | Mapped | PRIME.CISOTRXNS.SOURCE | ❌ Raw source string; no canonical mapping |
| Channel Terminal Id | STRING | Mapped | PRIME.CINSTALMENTS.TERMINALID | ✅ Acceptable as STRING reference key |
| Channel Bank | STRING | Mapped | PRIME.CACCOUNTSTMT.BANKNAME | ❌ Embedded bank name — should be FK `reference_bank_branch_bk` |
| Channel Branch | STRING | Mapped | PRIME.CACCOUNTSTMT.BANKBRANCH | ❌ Embedded branch name — should be FK `reference_bank_branch_bk` |
| Channel User Id | STRING | Not Available | None | ❌ Unmapped critical audit field |
| Channel Action Code | STRING | Not Available | None | ❌ Unmapped; `reference_channel_action_code` entity has no physical table |
| Channel Reference Number | STRING | Not Available | None | ❌ Unmapped — required for payment reconciliation |
| Channel Response Result Code | STRING | Not Available | None | ❌ Unmapped |
| Channel Response Code | STRING | Not Available | None | ❌ Unmapped |
| Channel Response Code Desc | STRING | Not Available | None | ❌ Unmapped |
| Channel Response Extend Code | STRING | Not Available | None | ❌ Unmapped |
| Channel Response Extend Description | STRING | Not Available | None | ❌ Unmapped |
| Channel Transaction Type | STRING | Mapped | PRIME.CTRANSACTIONS.TRXNTYPE | ❌ Duplicate of `Transaction type` — same source column; potential redundancy |
| CIF Number | STRING | Mapped | PRIME.CACCOUNTS.CUSTSERNO | ⚠️ FK reference to Party SA by raw string; no `party_sk` FK |
| Channel Id | STRING | Not Available | None | ❌ Unmapped; should be FK to `reference_channel` |
| Card Number Or Access Id | STRING | Mapped | PRIME.CINSTALMENTS.CARDNUMBER | ❌ PII — card number must be SHA-256 masked per Gate 2 controls |
| Channel Transaction Amount | STRING | Mapped | PRIME.CTRANSACTIONS.I004_AMT_TRXN | ❌ Must be DECIMAL(18,4); monetary amounts in fils for AED (SLV-010) |
| Channel Currency Code | STRING | Mapped | PRIME.CTRANSACTIONS.I049_CUR_TRXN | ❌ Raw ISO code; FK to `reference_currency` not documented |
| Channel Transaction Amount Currency Converter | STRING | Not Available | None | ❌ Unmapped |
| Channel Currency Converter Code | STRING | Not Available | None | ❌ Unmapped |
| Service Charge | STRING | Mapped | PRIME.SERVICES.SERVICEAMOUNT | ❌ Monetary — must be DECIMAL(18,4); naming should follow `<measure>_amount_aed` |
| Expiry Date | STRING | Mapped | PRIME.CARDX.EXPIRYDATE | ❌ Should be DATE type |
| From Account Number | STRING | Not Available | None | ❌ Unmapped critical debit-side reference |
| From Account Type | STRING | Not Available | None | ❌ Unmapped |
| To Account Number | STRING | Not Available | None | ❌ Unmapped critical credit-side reference |
| To Account Type | STRING | Not Available | None | ❌ Unmapped |
| Transaction In Time | STRING | Not Available | None | ❌ Unmapped; should be TIMESTAMP UTC |
| Transaction Out Time | STRING | Not Available | None | ❌ Unmapped |
| Card Acceptor Name | STRING | Not Available | None | ❌ Unmapped |
| Transaction Expected Out Time | STRING | Not Available | None | ❌ Unmapped |
| Channel Cross Border | STRING | Not Available | None | ❌ Unmapped; UAE cross-border flag important for CBUAE reporting |
| Channel Message | STRING | Not Available | None | ❌ Unmapped |
| Approved By | STRING | Mapped | PRIME.CISOTRXNS.I038_AUTH_ID | ✅ Auth ID acceptable |
| Posting Request Id | STRING | Mapped | PRIME.CISOTRXNS.I011_TRACE_NUM | ✅ Trace number |
| Financial Id | STRING | Mapped | PRIME.CISOTRXNS.SERNO | ✅ Serial number |
| Segment | STRING | Mapped | PRIME.CTRANSACTIONS.OFTRXNTYPE | ❌ Raw source code; must use code mapping |
| Segment Description | STRING | Mapped | PRIME.CTRANSACTIONS.OFRECTYPE | ❌ Pre-resolved description violates code mapping principle |
| Transaction Time | STRING | Mapped | PRIME.CISOTRXNS.I012_TRXN_TIME | ❌ Should be TIMESTAMP UTC |
| From Customer Number | STRING | Not Available | None | ❌ Unmapped |
| To Customer Number | STRING | Mapped | PRIME.CISOTRXNS.I032_ACQUIRER_ID | ⚠️ Acquirer ID used as "To Customer Number" — semantic mapping suspect |
| Source System Code | STRING | Mapped | None (derived) | ✅ |
| Source System Id | STRING | Mapped | None (derived) | ✅ |
| IsLogicallyDeleted | STRING | Not Available | None | ❌ Should be `delete_date TIMESTAMP` (nullable) |
| Create Date | STRING | Mapped | None | ❌ Should be `create_date TIMESTAMP` |
| Update Date | STRING | Mapped | None | ❌ Should be `update_date TIMESTAMP` |
| Deleted Date | STRING | Not Available | None | ❌ Should be `delete_date TIMESTAMP` (duplicate of IsLogicallyDeleted) |
| Business Start Date | CHAR(18) | N/A | N/A | ❌ Must be `effective_from TIMESTAMP` (SLV-003) |
| Business End Date | CHAR(18) | N/A | N/A | ❌ Must be `effective_to TIMESTAMP` (SLV-003) |
| Is Active Flag | CHAR(18) | N/A | N/A | ❌ Must be `is_current BOOLEAN` (SLV-003) |

#### 4.1c Metadata Completeness (SLV-008 / Section 5 of guidelines/01-data-quality-controls.md)

| Required Column | Present? | Notes |
|---|---|---|
| `record_source` / `source_system_code` | ✅ | Mapped (no source column; derived) |
| `record_origin` | ❌ | Absent |
| `record_hash` | ❌ | Absent |
| `is_current` | ❌ | Wrong column name and type (`Is Active Flag` CHAR(18)) |
| `valid_from_at` / `effective_from` | ❌ | Wrong column name and type (`Business Start Date` CHAR(18)) |
| `valid_to_at` / `effective_to` | ❌ | Wrong column name and type (`Business End Date` CHAR(18)) |
| `ingested_at` / `source_ingestion_date` | ❌ | Absent |
| `processed_at` / `create_date` | ⚠️ | Present but STRING instead of TIMESTAMP |
| `batch_id` / `run_id` | ❌ | Absent |
| `data_quality_status` / `_dq_status` | ❌ | Absent |
| `dq_issues` / `_dq_flags` | ❌ | Absent |

**Metadata score: 1/11 fully compliant. Critical gap.**

---

### Finding ECT-001 — Missing Surrogate and Business Key

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | `SLV-001` — "Every Silver entity must have a deterministic MD5 surrogate key `<entity>_sk`" · `SLV-002` — "Every Silver entity must have a tagged business key `<entity>_bk`" |
| Evidence | Logical model: `Txn - Electronic Channel Txn → Electronic Channel Transaction` — no `_sk` or `_bk` attribute · Physical: `electronic_channel_transaction` table has no SK column |
| Affected Table | `silver.transaction.electronic_channel_transaction` |
| Affected Column(s) | `electronic_channel_transaction_sk` (missing), `electronic_channel_transaction_bk` (missing) |
| Confidence | 0.97 |

**Description:** No surrogate key exists. The entity uses raw `Transaction Id` (PRIME.CISOTRXNS.I062V2_TRANS_ID) as its primary identifier but this is never tagged as `_bk`. Without a deterministic MD5 SK, pipeline re-runs generate no consistent downstream FK linkage to Gold fact tables. SCD-2 MERGE is impossible without an SK.

**Remediation:** Add `electronic_channel_transaction_sk STRING NOT NULL` (= `MD5(UPPER(TRIM(transaction_id)))`) and rename/tag `Transaction Id` as `electronic_channel_transaction_bk`. Apply identical pattern to all transaction entities.

**Estimated Effort:** Medium  
**Owner:** Data Modeling / ETL

---

### Finding ECT-002 — All Attributes Typed STRING

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | `SLV-010` — "Monetary amounts stored in smallest currency unit (fils for AED)" · `SLV-009` — "All timestamps stored in UTC" · Section 4, DQ Controls — "Monetary columns must use DECIMAL(18,4)" |
| Evidence | Logical model: every attribute for this entity shows type STRING · Data mapping: `Channel Transaction Amount` mapped as STRING from PRIME.CTRANSACTIONS.I004_AMT_TRXN |
| Affected Table | `silver.transaction.electronic_channel_transaction` |
| Affected Column(s) | `channel_transaction_amount`, `service_charge`, `channel_transaction_date`, `transaction_time`, `expiry_date`, `create_date`, `update_date` |
| Confidence | 0.97 |

**Description:** Every attribute including monetary amounts, dates, timestamps, and boolean flags is typed STRING. This means: (1) no range validation, (2) no numeric precision for amounts leading to rounding errors in regulatory reports, (3) date comparisons are string comparisons (incorrect for value-date arithmetic), (4) UTC normalisation cannot be enforced on a STRING column. This is a BCBS 239 risk-data lineage failure.

**Remediation:** `channel_transaction_amount` → `BIGINT` (fils/smallest unit, or `DECIMAL(18,4)` as per DQ guideline); `channel_transaction_date` → `DATE`; `transaction_time` → `TIMESTAMP NOT NULL`; `expiry_date` → `DATE`; `create_date`/`update_date` → `TIMESTAMP`; `effective_from`/`effective_to` → `TIMESTAMP`; `is_current` → `BOOLEAN`.

**Estimated Effort:** Large  
**Owner:** Data Modeling / ETL / DBA

---

### Finding ECT-003 — Raw Source Codes Stored Without Canonicalization

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | High |
| Guideline Rule | `SLV-004` — "No source-system codes; use `silver.reference.code_mapping`" · Modeling dos/donts §2.5 — "Resolve all source-system codes to canonical platform values" |
| Evidence | Data mapping: `Transaction type` Transformation_Logic = "Straight through field" from PRIME.CTRANSACTIONS.TRXNTYPE · `Channel Transaction Source` = Straight through from PRIME.CISOTRXNS.SOURCE · `Segment` = Straight through from PRIME.CTRANSACTIONS.OFTRXNTYPE |
| Affected Table | `silver.transaction.electronic_channel_transaction` |
| Affected Column(s) | `transaction_type`, `channel_transaction_source`, `segment`, `segment_description` |
| Confidence | 0.95 |

**Description:** Four attributes are passed straight from PRIME without canonical code resolution. PRIME-specific codes (internal numeric/abbreviated values) will be stored in Silver, making the data meaningless to any consumer not familiar with the PRIME schema. Also, `segment_description` (a pre-resolved description string) violates SLV-006 (no derived values).

**Remediation:** Join to `silver.reference.code_mapping` with `source_system = 'PRIME'`, `source_domain = 'TRANSACTION_TYPE'` etc. Drop `segment_description` (derived from code; resolve at Gold).

**Estimated Effort:** Medium  
**Owner:** ETL / Data Modeling

---

### Finding ECT-004 — PII Card Number Unmasked

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | High |
| Guideline Rule | Gate 2 PII Masking — "Card Numbers masked via SHA-256 unless stored in a restricted container" (guidelines/01-data-quality-controls.md §2, Gate 2) |
| Evidence | Data mapping: `Card Number Or Access Id` Transformation = "Straight through field" from PRIME.CINSTALMENTS.CARDNUMBER |
| Affected Table | `silver.transaction.electronic_channel_transaction` |
| Affected Column(s) | `card_number_or_access_id` |
| Confidence | 0.95 |

**Description:** Card number is mapped as a straight-through field. Under the Gate 2 rule, card numbers must be SHA-256 hashed before entering Silver unless the table is in a restricted data container with appropriate access controls. No restricted container is indicated. This is a potential PCI-DSS and UAE PDPL compliance gap.

**Remediation:** Apply `SHA2(UPPER(TRIM(card_number)), 256)` in the Silver pipeline transform. Retain raw value only in Bronze.

**Estimated Effort:** Small  
**Owner:** ETL / Data Governance

---

### Finding ECT-005 — 26 Attributes Unmapped

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | High |
| Guideline Rule | `SLV-005` — "DQ-gated writes; quarantine table required" — unmapped mandatory fields cause silent null propagation |
| Evidence | Data mapping: `Txn_Elect_Channel_Data_Model` sheet — 26 of 50 attributes marked "Not Available" including `Channel User Id`, `Channel Reference Number`, `From Account Number`, `To Account Number`, `Transaction In Time` |
| Affected Table | `silver.transaction.electronic_channel_transaction` |
| Affected Column(s) | All 26 "Not Available" attributes |
| Confidence | 0.97 |

**Description:** Mapping completeness is 48%. Critically, `From Account Number`, `To Account Number`, `Channel Reference Number` (needed for payment reconciliation), `Transaction In Time`/`Out Time` (needed for latency SLAs), and `Channel Cross Border` (needed for CBUAE cross-border reporting) are all unmapped. The entity cannot support regulatory reporting in its current state.

**Remediation:** Prioritize mapping for `from_account_number`, `to_account_number`, `channel_reference_number`, `channel_cross_border` in next sprint. Mark null-tolerant operational fields (P2) for next backlog cycle.

**Estimated Effort:** Large  
**Owner:** ETL / Source System SME

---

### Finding ECT-006 — Missing Complete Metadata Column Set

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | Medium |
| Guideline Rule | `SLV-008` — "All metadata columns present" · guidelines/11-modeling-dos-and-donts.md §2.7 — "Mandatory Silver Technical Audit Columns" |
| Evidence | Logical model: attributes present — `source_system_code`, `create_date`, `update_date`; absent — `record_hash`, `batch_id`/`run_id`, `source_ingestion_date`, `data_quality_status`, `dq_issues`; wrong type — `Business Start Date` CHAR(18) |
| Affected Table | `silver.transaction.electronic_channel_transaction` |
| Affected Column(s) | `record_hash`, `run_id`, `source_ingestion_date`, `data_quality_status`, `dq_issues` |
| Confidence | 0.92 |

**Description:** Six of eleven required metadata/audit columns are absent or incorrectly typed. Without `data_quality_status` and `dq_issues`, DQ-gate-2 results cannot be tracked against this entity. Without `record_hash`, deduplication logic has no basis. Note: the guidelines/01-data-quality-controls.md and guidelines/11-modeling-dos-and-donts.md use slightly different column name sets; see Appendix B for reconciliation recommendation.

**Remediation:** Add all missing metadata columns using canonical names from §2.7 of modeling guidelines. Resolve naming conflict between `record_source`/`source_system_code` at team level (see §8.B).

**Estimated Effort:** Medium  
**Owner:** Data Modeling / DBA

---

### 4.2 Internet/Mobile Banking Transaction

**Physical Table:** `silver_dev.electronic_channel_transaction.internet_mobile_banking_transaction` (12.6 MB)  
**Logical Model:** `Txn - Electronic Channel Txn → Internet/Mobile Banking Transaction` (47 attributes)  
**Mapping Completeness:** 100% (all 44 mapped in the data mapping workbook)

#### 4.2a Industry Fitness

This entity captures FLEXCUBE digital banking transactions (Internet + Mobile). It is appropriately scoped to digital channel interactions. The grain is one row per digital transaction.

**Concerns:**
- `Beneficiary Name` and `Merchant Name` are embedded in the transaction record — these are Party/Merchant SA attributes that create cross-SA denormalization. They will be stale if party name changes.
- `User Agent` is present — appropriate for digital fraud analytics.
- `Financial Indicator` (boolean type expected) and `Customer Flag` are STRING — should be BOOLEAN.
- The entity correctly separates From/To account references.

#### 4.2b Attribute-Level Review (selected critical items)

| Attribute | Issue |
|---|---|
| Transaction Id | ❌ No `_sk`/`_bk` pattern |
| Transaction Amount | ❌ STRING — must be DECIMAL(18,4) |
| Transaction Date | ❌ STRING — must be DATE |
| Transaction Time | ❌ STRING — must be TIMESTAMP UTC |
| Currency Code | ❌ Raw ISO code; no FK to `reference_currency` |
| Beneficiary Name | ❌ Cross-SA denormalization (Party SA) |
| Merchant Name | ❌ Cross-SA denormalization (Merchant entity) |
| Merchant id (MID) | ⚠️ FK to `reference_merchant` but not documented as FK |
| Response Code | ❌ Raw source code; needs canonical mapping |
| Financial Indicator | ❌ STRING — must be BOOLEAN |
| Business Start/End Date | ❌ CHAR(18) — must be `effective_from/to TIMESTAMP` |
| Is Active Flag | ❌ CHAR(18) — must be `is_current BOOLEAN` |
| IsLogicallyDeleted | ❌ STRING — must be `delete_date TIMESTAMP` nullable |

#### 4.2c Metadata Completeness

Same gaps as ECT-006 apply. No `record_hash`, `run_id`, `source_ingestion_date`, `data_quality_status`, `dq_issues`.

---

### Finding IMB-001 — Denormalized Party Attributes

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | Medium |
| Guideline Rule | `SLV-006` — "3NF; no derived values/aggregations" · Modeling dos/donts §2.2 — "Subject-Area Isolation" |
| Evidence | Data mapping: `Beneficiary Name` and `Merchant Name` both typed STRING and present in this transactional entity |
| Affected Table | `silver.transaction.internet_mobile_banking_transaction` |
| Affected Column(s) | `beneficiary_name`, `merchant_name` |
| Confidence | 0.90 |

**Description:** `Beneficiary Name` and `Merchant Name` are party-level attributes that belong in `silver.party` and `silver.transaction.reference_merchant` respectively. Embedding them violates 3NF (transitive dependency on party identifier) and means that name changes in the Party system do not propagate to transaction history correctly — a data consistency gap. These columns should only be `beneficiary_bk` (business key reference) and `merchant_id`.

**Remediation:** Replace `beneficiary_name`/`merchant_name` columns with `beneficiary_bk STRING` (FK reference to Party SA) and `merchant_bk STRING` (FK to reference_merchant). Name resolution performed at Gold layer.

**Estimated Effort:** Medium  
**Owner:** Data Modeling / ETL

---

### Finding IMB-002 — Missing Surrogate Key (identical to ECT-001)

Applies equally — see ECT-001. `internet_mobile_banking_transaction_sk` and `internet_mobile_banking_transaction_bk` both absent.

---

### 4.3 Payment Platform Transaction

**Physical Table:** `silver_dev.electronic_channel_transaction.payment_platform_transaction` (2.9 GB, 24 files)  
**Logical Model:** `Txn - Electronic Channel Txn → Payment Platform Transaction` (47 attributes)  
**Mapping Completeness:** 77% (34 of 44 mapped)  
**Delta Format Note:** This table was created 2025-10-09 (older than other tables, Oct vs Dec 2025) and lacks `delta.parquet.compression.codec` and `delta.autoOptimize.optimizeWrite` — it predates the standardised Delta table configuration applied to other tables.

#### 4.3a Industry Fitness

This entity models payment platform messages — it resembles an ISO 20022 pacs.008/pacs.009 message record with debtor and creditor parties. The grain is one payment platform transaction.

**Critical concerns:**
- **Embedded debtor/creditor customer names and segment codes**: `Creditor Customer Name`, `Debtor Customer Name`, `Creditor Segment Code`, `Debtor Segment Code` are directly embedded. These are Party SA attributes and violate 3NF.
- **Multiple duplicate mappings**: `Transaction type`, `Transaction code`, `Transaction category`, `Financial transaction type`, `Platform transaction category` all appear to derive from `PRIME.CTRANSACTIONS.TRXNTYPE` — the same source column is being loaded into 4 different logical attributes without clear semantic differentiation.
- **`Creditor Customer Number` mapped to `PRIME.CISOTRXNS.I042_MERCH_ID`** (Merchant ID) — this is semantically incorrect; acquiring merchant ID ≠ creditor customer number. High-risk incorrect mapping.

#### 4.3b Selected Critical Attribute Issues

| Attribute | Mapping Source | Issue |
|---|---|---|
| Transaction type | PRIME.CISOTRXNS.AUTHTRXNTYPE | Raw source code; needs canonicalization |
| Transaction code | PRIME.CTRANSACTIONS.TRXNTYPE | Same source as `financial_transaction_type` — duplicate |
| Transaction Status | PRIME.CTRANSACTIONS.I048_TEXT_DATA | `I048_TEXT_DATA` is a freetext field — mapping transaction status to freetext is incorrect |
| Transaction Amount | STRING | Must be DECIMAL(18,4) |
| Transaction Fee Amount | Same source as `Transaction Amount` (I004_AMT_TRXN) | Ambiguous — fee amount ≠ transaction amount; wrong mapping candidate |
| Creditor Customer Number | PRIME.CISOTRXNS.I042_MERCH_ID | **Incorrect** — Merchant ID ≠ Customer Number |
| Debtor Customer Number | PRIME.CISOTRXNS.I042_MERCH_ID | **Incorrect** — same Merchant ID used for both Creditor and Debtor customer numbers |
| Transaction Time | PRIME.CTRANSACTIONS.I007_LOAD_DATE | LOAD_DATE used as transaction time — this is system load time, not transaction time |
| Creditor/Debtor Customer Name | Embedded | ❌ Cross-SA denormalization; Party SA attribute |
| Creditor/Debtor Segment Code | Embedded | ❌ Cross-SA denormalization; Party SA attribute |

---

### Finding PPT-001 — Incorrect Mapping: Creditor/Debtor Customer Number → Merchant ID

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | Behavioral rule — "Do not assume correct mapping based on similar column names — verify type, nullability, and semantics" |
| Evidence | Data mapping: `Creditor Customer Number` → PRIME.CISOTRXNS.I042_MERCH_ID AND `Debtor Customer Number` → PRIME.CISOTRXNS.I042_MERCH_ID (same source column for both) |
| Affected Table | `silver.transaction.payment_platform_transaction` |
| Affected Column(s) | `creditor_customer_number`, `debtor_customer_number` |
| Confidence | 0.92 |

**Description:** Both `Creditor Customer Number` and `Debtor Customer Number` are mapped to the same source column `I042_MERCH_ID` (ISO 8583 field 42 = Merchant ID). This is semantically incorrect — a merchant ID is not a customer number. In a payment context, the debtor customer number should come from the initiating account holder, and the creditor from the beneficiary/payee. Using merchant ID for both will produce incorrect data in any payment analytics use case.

**Remediation:** Investigate correct source for customer numbers. Likely candidates: `PRIME.CCUSTOMERS.SERNO` (debtor), `PRIME.CCUSTOMERS.CUSTSERNO` or external bank BIC for creditor in cross-bank scenarios. Mark as ambiguous pending source system SME confirmation.

**Estimated Effort:** Medium  
**Owner:** ETL / Source System SME

---

### Finding PPT-002 — Transaction Status Mapped to Free-Text Field

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | High |
| Guideline Rule | `SLV-004` — "No source-system codes; use `silver.reference.code_mapping`" · Gate 2 — "Domain/value-set validation applied" |
| Evidence | Data mapping: `Transaction status` → PRIME.CTRANSACTIONS.I048_TEXT_DATA (straight through) |
| Affected Table | `silver.transaction.payment_platform_transaction` |
| Affected Column(s) | `transaction_status` |
| Confidence | 0.90 |

**Description:** ISO 8583 Field 48 (`I048_TEXT_DATA`) is an additional data / narrative freetext field, not a status code. Mapping transaction status to this field will produce unstructured text values instead of a controlled vocabulary status code. This breaks DQ domain validation and any status-based reporting (e.g., count of SETTLED vs PENDING vs REVERSED transactions).

**Remediation:** Map `transaction_status` to the appropriate PRIME status field (likely `PRIME.CTRANSACTIONS.TRXNTYPE` with a status discriminator or a dedicated status table). Confirm with PRIME system SME.

**Estimated Effort:** Small (once correct source identified)  
**Owner:** ETL / Source System SME

---

### Finding PPT-003 — Delta Table Not Upgraded to Current Standard Configuration

| Field | Value |
|---|---|
| Priority | P2 |
| Criticality | Low |
| Guideline Rule | Platform operational standard (no specific guideline rule; best practice) |
| Evidence | Physical structures CSV: `payment_platform_transaction` lacks `delta.parquet.compression.codec=zstd`, `delta.autoOptimize.autoCompact=true`, `delta.autoOptimize.optimizeWrite=true` properties present on all other tables |
| Affected Table | `silver_dev.electronic_channel_transaction.payment_platform_transaction` |
| Confidence | 0.88 |

**Description:** This table (2.9 GB, largest in the SA) is missing the standard Delta table optimizations applied to all newer tables. Without `autoOptimize`, small files accumulate degrading scan performance. Without ZSTD compression, storage cost is higher.

**Remediation:** Run `ALTER TABLE payment_platform_transaction SET TBLPROPERTIES ('delta.parquet.compression.codec' = 'zstd', 'delta.autoOptimize.autoCompact' = 'true', 'delta.autoOptimize.optimizeWrite' = 'true')` then run `OPTIMIZE` on the table.

**Estimated Effort:** Small  
**Owner:** DBA / Platform Engineering

---

### 4.4 Non Electronic Channel Transaction

**Physical Table:** `silver_dev.non_electronic_channel_transaction.non_electronic_channel_transaction` (4.5 GB, 2 files)  
**Logical Model:** `Txn - NonElectronic Channel Txn → Non Electronic Channel Transaction` (26 attributes)  
**Mapping Completeness:** 78%

#### 4.4a Industry Fitness

This entity acts as the **header/parent** record for all non-electronic channel transactions (ATM, Branch, POS, Phone Banking). It captures the core financial movement: amount, currency, account, customer, date, status. Sub-channel entities (ATM, Branch, POS, PhoneBanking) extend this with channel-specific attributes via FK on `Transaction id`.

The grain is one transaction record per channel event — this is appropriate. The entity carries `Value date` and `Balance after transaction` which is notable.

**`Balance after transaction` is a derived/aggregated value** — it is not an atomic fact but a running balance at the point of posting. This violates SLV-006 (no derived values in Silver). Balance should be in the Account SA, not embedded in each transaction row.

#### 4.4b Attribute Issues

| Attribute | Issue |
|---|---|
| Transaction id | ❌ No `_sk`/`_bk` |
| Amount | ❌ STRING; must be DECIMAL(18,4) |
| Transaction date | ❌ STRING; must be DATE |
| Transaction time | ❌ STRING; must be TIMESTAMP UTC |
| Value date | ❌ CHAR(18) in model; must be DATE |
| Balance after transaction | ❌ CHAR(18); derived value — violates SLV-006 |
| Exchange rate | ❌ STRING; must be DECIMAL(18,6) |
| Channel type | ❌ Raw source code; needs `code_mapping` |
| Sub Channel | ❌ Not Available; no `reference_sub_channel` entity |
| Transaction type | ❌ Raw PRIME code |
| Status | ❌ Mapped to PRIME.CINSTALMENTS.STATUS — wrong source table (CINSTALMENTS is instalment/loan table, not transaction status) |
| IsLogicallyDeleted | ❌ Not Available; should be `delete_date TIMESTAMP` |

---

### Finding NECT-001 — Balance After Transaction Violates SLV-006

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | High |
| Guideline Rule | `SLV-006` — "3NF; no derived values/aggregations" · `SLV-007` — "No pre-computed metrics" |
| Evidence | Logical model: `Balance after transaction CHAR(18)` mapped from PRIME.CTRANSACTIONS.I005_AMT_SETTLE |
| Affected Table | `silver.transaction.non_electronic_channel_transaction` |
| Affected Column(s) | `balance_after_transaction` |
| Confidence | 0.88 |

**Description:** `Balance after transaction` is a running balance — a derived aggregate computed from the sum of all prior transaction amounts on the account. Storing this in a transaction entity violates SLV-006/SLV-007. It also creates an implicit dependency on account state, breaking subject-area isolation. Account balances belong in `silver.account.account_balance`.

**Remediation:** Remove `balance_after_transaction` from this entity. If post-posting balance is needed for audit purposes, store it in `silver.account.account_balance_snapshot` or document explicit steward approval for the exception.

**Estimated Effort:** Small (column removal)  
**Owner:** Data Modeling / Data Steward

---

### Finding NECT-002 — Transaction Status Mapped to Wrong Source Table

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | Medium |
| Guideline Rule | Assessment rule — "Verify type, nullability, and semantics; do not assume correct mapping based on column names" |
| Evidence | Data mapping: `Status` → PRIME.CINSTALMENTS.STATUS (straight through) |
| Affected Table | `silver.transaction.non_electronic_channel_transaction` |
| Affected Column(s) | `transaction_status` |
| Confidence | 0.85 |

**Description:** `PRIME.CINSTALMENTS` is the instalment/scheduled payment table for card instalments, not the primary transaction table. Using its `STATUS` column for all non-electronic channel transactions is semantically incorrect — branch withdrawals, ATM transactions, and POS transactions do not have instalment status. The correct source is likely `PRIME.CTRANSACTIONS.TRXNTYPE` combined with a status field or `PRIME.CISOTRXNS.I039_RESP_CD` (ISO 8583 response code).

**Remediation:** Confirm correct transaction status source with PRIME system SME. Likely `PRIME.CTRANSACTIONS` has a status indicator.

**Estimated Effort:** Small  
**Owner:** ETL / Source System SME

---

### 4.5 Non Electronic Channel Transaction (ATM)

**Physical Table:** `silver_dev.non_electronic_channel_transaction.non_electronic_channel_transaction_atm` (2.5 GB, 2 files)  
**Logical Model:** ATM (17 attributes)  
**Mapping Completeness:** 57%

#### 4.5a Industry Fitness

ATM transactions extend the parent Non Electronic Channel Transaction via `Transaction id` FK. They add ATM-specific fields: card number, action, card type, auth code, response code.

**Missing:** `ATM location` is Not Available — this is critical for ATM network analytics and CBUAE reporting. The `Reference_ATM` entity (14 attrs) has no physical implementation — ATM master data is unavailable.

#### 4.5b Key Issues

| Attribute | Issue |
|---|---|
| Transaction id | ❌ No `_sk`/`_bk` |
| Channel Id | ❌ Not Available — FK to which channel is missing |
| Card number | ❌ PII — must be SHA-256 masked |
| Card type | ❌ Raw source code |
| ATM location | ❌ Not Available — no `reference_atm` physical table |
| Action | ❌ Raw source code (LOGACTION — log action code) |

---

### Finding ATM-001 — Reference_ATM Entity Has No Physical Implementation

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | High |
| Guideline Rule | `SLV-005` — "DQ-gated writes" requires referential integrity on ATM FK · Modeling §2.11 — "Erwin model before DDL" |
| Evidence | Physical structures CSV: no `reference_atm` table exists in either schema · Logical model: `Reference_ATM` with 14 attributes (Channel Id, ATM Id, ATM Serial Num, etc.) all "Not Available" |
| Affected Table | `silver.transaction.reference_atm` (missing) |
| Affected Column(s) | All 14 |
| Confidence | 0.97 |

**Description:** ATM master data entity exists in the logical model but has no physical table. This means ATM location, model, and channel type cannot be resolved for ATM transactions. CBUAE requires ATM-level reporting for network statistics. The `ATM location` attribute in the ATM transaction entity is also unmapped for the same reason.

**Remediation:** Implement `reference_atm` table. Source: check if PRIME has an ATM master table or if the Organization SA (`silver.org.atm`) is the correct cross-SA reference. Note: ATM master belongs in `silver.org` per `sa/subject-areas.md` §18 ("ATMs" listed under Organization SA).

**Estimated Effort:** Medium  
**Owner:** Data Modeling / ETL / Organization SA team

---

### 4.6 Non Electronic Channel Transaction (Branch)

**Physical Table:** `silver_dev.non_electronic_channel_transaction.non_electronic_channel_transaction__branch_` (5.2 KB — very small, potentially incomplete)  
**Logical Model:** Branch (20 attributes)  
**Mapping Completeness:** 35% (6 of 17 mapped)

#### 4.6a Industry Fitness

Branch transactions extend the parent with teller, counter, queue, and ID verification attributes. The small physical size (5.2 KB) is suspicious given the scale of the bank — this may indicate a pipeline loading issue or test data only.

**Key gaps:** `Cheque number`, `ID proof type`, `ID`, `Transaction purpose` all unmapped. These are critical for AML transaction monitoring (purpose and ID verification are FATF requirements) and for cheque clearing reconciliation.

---

### Finding BRANCH-001 — Table Naming Violation and Critically Low Mapping

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | High |
| Guideline Rule | `NAM-001` — "snake_case, lowercase, no reserved words" · `NAM-003` — "Silver entity: `<entity>`" |
| Evidence | Physical: table name `non_electronic_channel_transaction__branch_` contains double underscores and leading/trailing underscores around `branch` |
| Affected Table | `silver_dev.non_electronic_channel_transaction.non_electronic_channel_transaction__branch_` |
| Confidence | 0.99 |

**Description:** Double underscores and the pattern `__branch_` are invalid per NAM-001 (snake_case). The table should be named `non_electronic_channel_transaction_branch`. Additionally, at 35% mapping completeness, AML-critical fields (`transaction_purpose`, `id_proof_type`, `id`) are unmapped.

**Remediation:** Rename table to `non_electronic_channel_transaction_branch` via breaking-change DDL migration (NAM-006 versioning procedure). Complete mappings for `cheque_number`, `id_proof_type`, `transaction_purpose`.

**Estimated Effort:** Medium  
**Owner:** DBA (rename) / ETL (mapping) / AML Compliance (requirements)

---

### 4.7 Non Electronic Channel Transaction (POS)

**Physical Table:** `silver_dev.non_electronic_channel_transaction.non_electronic_channel_transaction__pos_` (2.8 MB)  
**Logical Model:** POS (26 attributes with triplication artefact)  
**Mapping Completeness:** 71%

#### 4.7a Industry Fitness

POS transactions are appropriate here. MCC code, Merchant ID, Terminal ID, Auth code are the core attributes.

**Key issue:** The logical model has **triplicate** copies of `Business Start Date`, `Business End Date`, `Is Active Flag` with numeric suffixes (`__9057` through `__9065`) — this is a clear Erwin copy-paste error creating 9 duplicate SCD columns.

---

### Finding POS-001 — Duplicate SCD Columns in Logical Model (Erwin Artefact)

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | Medium |
| Guideline Rule | `SLV-003` — "SCD-2 default; history tracking" |
| Evidence | Logical model: `Non Electronic Channel Transaction (POS)` has `Business Start Date`, `Business End Date`, `Is Active Flag` x3 with suffixes `__9057-9065` |
| Affected Table | `silver.transaction.non_electronic_channel_transaction_pos` |
| Affected Column(s) | All 9 triplicated columns |
| Confidence | 0.99 |

**Description:** The Erwin model for the POS entity contains three sets of identical SCD columns, likely created by multiple copy operations without deduplication. If these are implemented as-is in DDL, the table will have 9 redundant SCD columns alongside the real ones. This must be corrected in the Erwin model before DDL generation.

**Remediation:** Delete duplicate `Business Start Date__9057/60/63`, `Business End Date__9058/61/64`, `Is Active Flag__9059/62/65` from the Erwin model. Retain one set, rename to `effective_from`, `effective_to`, `is_current`.

**Estimated Effort:** Small  
**Owner:** Data Modeling

---

### Finding POS-002 — Table Naming Violation

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | Low |
| Guideline Rule | `NAM-001` — snake_case · `NAM-003` — Silver entity naming |
| Evidence | Physical: `non_electronic_channel_transaction__pos_` |
| Affected Table | `silver_dev.non_electronic_channel_transaction.non_electronic_channel_transaction__pos_` |
| Confidence | 0.99 |

**Description:** Same pattern as BRANCH-001. Table should be `non_electronic_channel_transaction_pos`.

**Remediation:** Rename per breaking-change migration procedure.

**Estimated Effort:** Small  
**Owner:** DBA

---

### 4.8 Non Electronic Channel Transaction (PhoneBanking)

**Physical Table:** Served by parent `non_electronic_channel_transaction` (no distinct phone banking table)  
**Logical Model:** PhoneBanking (20 attributes)  
**Mapping Completeness:** 6% (1 of 17 mapped)

#### 4.8a Industry Fitness

Phone banking transactions are a legitimate sub-channel. The entity correctly captures agent, agency, call type, call ID, verification information — essential for customer identity verification audit.

**Critical state:** This entity is functionally non-existent in Silver. Only `Service request id` is mapped. The `Reference_PhoneBankingAgent` physical table is empty (0 bytes). There is no distinct physical table for phone banking sub-channel transactions (they likely fall through to the parent `non_electronic_channel_transaction` table without sub-channel extension rows).

---

### Finding PBK-001 — Phone Banking Entity Critically Unmapped (6%)

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | `SLV-005` — "DQ-gated writes; quarantine table" — missing data cannot be quality-gated · BCBS 239 — completeness requirement |
| Evidence | Data mapping: only `Service request id` mapped from `Prime.CISOTRXNS.TOKENREQUESTORID`; all other 16 attributes "Not Available" |
| Affected Table | `silver.transaction.non_electronic_channel_transaction_phonebanking` |
| Affected Column(s) | `transaction_id`, `agent_id`, `agency_id`, `channel_id`, `call_type`, `call_id`, `request_type`, `verified_flag`, `id_type`, `id` |
| Confidence | 0.97 |

**Description:** Phone banking transaction records land in the generic parent table without any agent, agency, call, or verification attributes. This means: (1) phone banking transaction volume cannot be attributed to agents, (2) customer identity verification during phone calls cannot be audited — an AML and KYC gap, (3) call type and request type are needed for contact centre analytics. The `reference_phonebankingagent` table is empty.

**Remediation:** (1) Identify source system for phone banking call records — this may require integration with the contact centre platform (not PRIME). (2) If PRIME does not hold phone banking agent data, source from the contact centre system. (3) Create a distinct physical table `non_electronic_channel_transaction_phonebanking` with the extension attributes. (4) Populate `reference_phonebankingagent` from HR/Organization system.

**Estimated Effort:** Large  
**Owner:** ETL / Source System SME / Business Event SA team

---

### 4.9 Reference_Bank_Branch

**Physical Tables:** Duplicated — `silver_dev.electronic_channel_transaction.reference_bank_branch` (34.5 KB) AND `silver_dev.non_electronic_channel_transaction.reference_bank_branch` (5.9 KB)  
**Logical Model Entities:** Two separate but identical `Reference_Bank_Branch` entities in both sub-areas (18 attrs each)  
**Mapping Completeness:** 56% (10 of 18 mapped)

#### 4.9a Industry Fitness

Bank branch master data is a Reference Data SA concern, not a Transaction SA concern. Per `sa/subject-areas.md`, Organization SA (`silver.org`) contains "Branches, Departments, Cost Centers, Regions, ATMs, and Physical Assets." Branch master data should sit in `silver.org` and be referenced from transaction entities by FK.

Having two physical copies (in electronic and non-electronic schemas) with different file sizes (34.5 KB vs 5.9 KB) strongly suggests they are out of sync — a data quality risk.

---

### Finding REFBR-001 — Reference_Bank_Branch Duplicated Across Schemas and Wrong SA

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | High |
| Guideline Rule | `SLV-002` — Subject-area isolation · `SLV-004` — use `silver.reference.code_mapping` for codes |
| Evidence | Physical structures: `reference_bank_branch` in BOTH `electronic_channel_transaction` (34.5 KB) and `non_electronic_channel_transaction` (5.9 KB) schemas |
| Affected Table | Both copies |
| Confidence | 0.95 |

**Description:** Branch master data duplicated across two schemas with different sizes — high probability they are diverged. Branch data belongs in either `silver.org.branch` (Organization SA) or `silver.reference` as a lookup, not in the transaction SA. Transaction entities should reference branches by FK.

**Remediation:** Migrate to `silver.org.branch` (canonical per Organization SA definition). Update transaction pipeline joins to use the Organization SA table. Drop both transaction-SA copies after migration.

**Estimated Effort:** Medium  
**Owner:** Data Modeling / Organization SA team

---

### 4.10 Reference_MCC (Merchant Category Code)

**Physical Tables:** `silver_dev.electronic_channel_transaction.reference_mcc` (0 bytes — EMPTY) and `silver_dev.non_electronic_channel_transaction.reference_mcc` (11.3 MB)  
**Mapping Completeness:** 69%

**Key Issue:** Electronic channel schema's `reference_mcc` is completely empty (0 bytes, 0 files). All MCC data resides only in the non-electronic schema. This means `electronic_channel_transaction` and `internet_mobile_banking_transaction` entities that reference MCC codes cannot resolve them through the local reference table.

---

### Finding REFMCC-001 — Reference_MCC Empty in Electronic Schema; Wrong SA

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | High |
| Guideline Rule | `SLV-005` — referential integrity | 
| Evidence | Physical: `electronic_channel_transaction.reference_mcc` — numFiles=0, sizeInBytes=0 |
| Affected Table | `silver_dev.electronic_channel_transaction.reference_mcc` |
| Confidence | 0.99 |

**Description:** The MCC reference table in the electronic schema has never been loaded. Card transactions using MCC codes (via `electronic_channel_transaction`) will have unresolvable MCC references. Additionally, having two MCC tables is redundant — both schemas use the same ISO 18245 MCC standard codes. Should be a single table in `silver.reference`.

**Remediation:** (1) Populate `electronic_channel_transaction.reference_mcc` immediately (or) migrate both to `silver.reference.mcc` as a shared table.

**Estimated Effort:** Small  
**Owner:** ETL / Reference Data steward

---

### 4.11 Reference_Merchant

**Physical Tables:** `silver_dev.electronic_channel_transaction.reference_merchant` (9.7 KB) and `silver_dev.non_electronic_channel_transaction.reference_merchant` (10.4 MB)  
**Mapping Completeness:** 75%

**Large size discrepancy** (9.7 KB vs 10.4 MB) between the two copies suggests significant divergence. The non-electronic copy (10.4 MB) appears to have substantially more merchant records than the electronic copy (9.7 KB). This is a data consistency risk.

---

### Finding REFMERC-001 — Reference_Merchant Diverged Across Schemas

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | High |
| Guideline Rule | Subject-area isolation; `SLV-006` 3NF |
| Evidence | Physical: electronic `reference_merchant` = 9.7 KB; non-electronic = 10.4 MB (~1000x difference) |
| Affected Table | Both copies |
| Confidence | 0.95 |

**Description:** Merchant master data diverged — the electronic schema copy is orders of magnitude smaller than the non-electronic copy. Merchant data belongs in a single table in `silver.reference` or (given the 28-attribute scope including financial terms) potentially its own `silver.payment.merchant` entity. Duplicate maintenance guarantees divergence over time. Also: `Contract end date` is mapped from `PRIME.MERCHANTS.PMNTCUTOFFTIME` — payment cutoff time ≠ contract end date, likely a semantic mapping error.

**Remediation:** Consolidate to `silver.reference.merchant`. Review `contract_end_date` ↔ `PMNTCUTOFFTIME` mapping with source system SME.

**Estimated Effort:** Medium  
**Owner:** Data Modeling / Reference Data steward

---

### 4.12 Reference_Transaction_Code

**Physical Table:** `silver_dev.electronic_channel_transaction.reference_transaction_code` (6.9 KB)  
**Mapping Completeness:** 83%  
**Note:** No corresponding table in non-electronic schema.

**Key concerns:**
- `Transaction amount limit`, `Transaction local amount`, `Transaction global amount` all mapped to `PRIME.MTRANSACTIONS.AMOUNT` (same column). These are three distinct concepts mapped to one source.
- `Ledger batch ID` (`TYPESERNO_GLEDGER`) is present — this is a GL reference, appropriate for transaction-to-GL linkage.
- `Risk level` mapped from `PRIME.MERCHANTS.RISKLEVEL` — risk level of a transaction code should come from a transaction code risk mapping, not a merchant table.

---

### Finding REFTXNCD-001 — Three Amount Attributes Mapped to Same Source Column

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | Medium |
| Guideline Rule | Assessment rule — verify semantics, not just column names |
| Evidence | Data mapping: `Transaction amount limit`, `Transaction local amount`, `Transaction global amount` all → PRIME.MTRANSACTIONS.AMOUNT |
| Affected Table | `silver.transaction.reference_transaction_code` |
| Affected Column(s) | `transaction_amount_limit`, `transaction_local_amount`, `transaction_global_amount` |
| Confidence | 0.90 |

**Description:** Three semantically distinct attributes — limit (cap), local amount (in originating currency), and global amount (in base currency) — all map to the same source column. At most one of these can be correct; the others are either incorrectly mapped or should have different sources (e.g., global amount may need FX-converted via `PRIME.CURRENCIES`).

**Remediation:** Investigate correct source for each: limit from a transaction type limit table; local amount from transaction record in originating currency; global amount from the AED-converted amount.

**Estimated Effort:** Small  
**Owner:** ETL / Source System SME

---

### 4.13–4.19 Channel Reference Entities (Reference_Channel, Reference_Channel_Type, Reference_Channel_Status, Reference_Channel_Category_Type)

| Entity | Physical Table | Size | Mapping % | Issue |
|---|---|---|---|---|
| Reference_Channel | `reference_channel` | 20.9 KB | **0%** | No mapping defined; entity attributes undocumented |
| Reference_Channel_Type | `reference_channel_type` | **0 bytes (EMPTY)** | **0%** | Table exists but never populated |
| Reference_Channel_Status | `reference_channel_status` | 7.5 KB | **0%** | Physical table populated but no mapping docs |
| Reference_Channel_Category_Type | `reference_channel_category_type` | 8.0 KB | **0%** | Physical table populated but no mapping docs |
| Reference_Currency | `reference_currency` | 6.5 KB | **82%** | Good progress; `Minor Unit` unmapped; correct SA = `silver.reference` |

---

### Finding REFCH-001 — Four Channel Reference Tables Have Zero Mapping Documentation

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | Medium |
| Guideline Rule | `SLV-005` — "DQ-gated writes" — undocumented mappings cannot be DQ-validated |
| Evidence | Data mapping workbook: no sheets cover Reference_Channel, Reference_Channel_Type, Reference_Channel_Status, Reference_Channel_Category_Type entities |
| Affected Tables | All four |
| Confidence | 0.95 |

**Description:** Four reference tables are physically present (three with data) but have zero ETL mapping documentation in the workbook. Without documented lineage, the source of their data is unknown, DQ validation is impossible, and audit trail is broken. `reference_channel_type` is also empty. All four belong in `silver.reference`.

**Remediation:** Document source-to-target mapping for all four entities. Populate `reference_channel_type`. Consider migration to `silver.reference`.

**Estimated Effort:** Medium  
**Owner:** ETL / Reference Data steward

---

### 4.20 Reference_Currency

**Physical Table:** `silver_dev.electronic_channel_transaction.reference_currency` (6.5 KB)  
**Mapping Completeness:** 82%

Primary concern is SA placement (should be in `silver.reference`) and the unmapped `Minor Unit` attribute (needed for fils/smallest unit conversion per SLV-010).

---

### 4.21 Reference_ATM (No Physical Table)

See Finding ATM-001 above.

---

### 4.22 Reference_PhoneBankingAgent (Empty Table)

**Physical Table:** `silver_dev.non_electronic_channel_transaction.reference_phonebankingagent` (0 bytes)  
**Logical Model:** 11 attributes (all "Not Available")  
**Mapping Completeness:** 27% (only `source_system_code`, `source_system_id`, `create_date` technically "mapped")

This entity belongs in `silver.org` (Organization SA — "Employees" or a contact centre agent entity). It is 0 bytes — no records ever loaded.

---

## 5. Denormalization Register

| # | Co-location | Entities | Classification | Justification Required |
|---|---|---|---|---|
| 1 | `Channel Bank` (bank name string) in `Electronic Channel Transaction` | ECT → Reference_Bank_Branch | **Unnecessary** | FK to `reference_bank_branch` should be used; name is a derived attribute |
| 2 | `Channel Branch` (branch name string) in `Electronic Channel Transaction` | ECT → Reference_Bank_Branch | **Unnecessary** | Same as above |
| 3 | `Segment Description` in `Electronic Channel Transaction` | ECT → code_mapping | **Unnecessary** | Pre-resolved description; violates SLV-004; resolve at Gold |
| 4 | `Creditor Customer Name` in `Payment Platform Transaction` | PPT → silver.party | **Unnecessary** | Party SA attribute; causes stale name data on name changes |
| 5 | `Debtor Customer Name` in `Payment Platform Transaction` | PPT → silver.party | **Unnecessary** | Same as above |
| 6 | `Creditor Segment Code` in `Payment Platform Transaction` | PPT → silver.party | **Unnecessary** | Party/segmentation attribute; cross-SA |
| 7 | `Debtor Segment Code` in `Payment Platform Transaction` | PPT → silver.party | **Unnecessary** | Same as above |
| 8 | `Beneficiary Name` in `Internet/Mobile Banking Transaction` | IMB → silver.party | **Unnecessary** | Stale on name change; FK reference preferred |
| 10 | `Merchant Name` in `Internet/Mobile Banking Transaction` | IMB → Reference_Merchant | **Acceptable** | Could keep for display convenience with documented justification; prefer FK |
| 11 | `Balance after transaction` in `Non Electronic Channel Transaction` | NECT → silver.account | **Unnecessary** | Derived metric; violates SLV-007; Account SA boundary |
| 12 | Reference data tables (`reference_mcc`, `reference_merchant`, `reference_bank_branch`) duplicated in BOTH electronic and non-electronic schemas | Both schemas | **Unnecessary** | Consolidate to `silver.reference`; duplicate maintenance is risk |

**Summary:** 11 of 12 co-locations are Unnecessary. Only `Merchant Name` in IMB has an arguable case for acceptable denormalization (display convenience) but requires documented data steward justification.

---

## 6. Guideline Compliance Summary

| Rule | Description | Status | Affected Entities |
|---|---|---|---|
| **SLV-001** | `<entity>_sk` MD5 surrogate on every entity | ❌ **Fail** — 0/18 entities compliant | All 18 |
| **SLV-002** | `<entity>_bk` business key on every entity | ❌ **Fail** — 0/18 entities compliant | All 18 |
| **SLV-003** | SCD-2 default | ❌ **Fail** — wrong column names (`Business Start/End Date` CHAR(18)) | All entities |
| **SLV-004** | No source-system codes; use `silver.reference.code_mapping` | ❌ **Fail** — multiple straight-through code mappings | ECT, PPT, NECT, IMB |
| **SLV-005** | DQ-gated writes; quarantine table | ⚠️ **Partial** — `data_quality_status`/`dq_issues` columns absent from all entities | All 18 |
| **SLV-006** | 3NF; no derived values/aggregations | ❌ **Fail** — `balance_after_transaction`, `segment_description`, party names embedded | NECT, ECT, PPT, IMB |
| **SLV-007** | No pre-computed metrics | ❌ **Fail** — `balance_after_transaction` | NECT |
| **SLV-008** | All metadata columns present | ❌ **Fail** — 6/11 required columns missing | All 18 |
| **SLV-009** | All timestamps UTC | ❌ **Fail** — all date/time columns are STRING; no UTC enforcement | All 18 |
| **SLV-010** | Monetary amounts in smallest unit (fils for AED) | ❌ **Fail** — all amounts are STRING; no fils denomination | All transaction entities |
| **NAM-001** | snake_case, lowercase, no reserved words | ❌ **Fail** — double-underscore table names (`__pos_`, `__branch_`) | NECT-Branch, NECT-POS |
| **NAM-002** | Catalog/schema naming pattern `silver.<subject_area>` | ❌ **Fail** — schemas are `electronic_channel_transaction` / `non_electronic_channel_transaction`, not `transaction` | All tables |
| **NAM-003** | Table naming `<entity>` for Silver | ⚠️ **Partial** — core tables follow pattern; reference tables should not be in this SA | Reference tables |
| **Gate 2 PII** | Card numbers masked via SHA-256 | ❌ **Fail** — `card_number` mapped as straight-through | ECT, NECT-ATM |
| **Gate 2 Completeness** | All required records for core entities | ❌ **Fail** — `reference_mcc` (electronic), `reference_channel_type`, `reference_phonebankingagent` empty; `reference_atm` missing | Multiple |
| **§2.4 Surrogate Keys** | MD5, not UUID/RAND | ❌ **Fail** — no SKs of any kind | All 18 |
| **§2.5 Code Canonicalization** | Resolve via code_mapping | ❌ **Fail** — multiple straight-through code mappings | ECT, PPT, NECT |
| **§2.7 Audit Columns** | Full mandatory column set | ❌ **Fail** — `record_hash`, `run_id`, `source_ingestion_date`, `data_quality_status`, `dq_issues` absent | All 18 |
| **BCBS 239** | Accurate, complete financial data | ❌ **Fail** — STRING types for monetary amounts; incomplete mappings; no DQ status | All transaction entities |

**Compliance Score: 0/19 rules fully compliant · 2/19 partially compliant · 17/19 failed**

---

## 7. Remediation Plan

### 7.1 Prioritized Action List

#### P0 — Immediate (current sprint)

| ID | Action | Rule | Owner | Effort |
|---|---|---|---|---|
| P0-1 | Rename schemas: create `silver.transaction` catalog; establish `silver.transaction.electronic_channel_transaction`, `silver.transaction.non_electronic_channel_transaction`, etc. as canonical tables | NAM-002 | DBA / Platform | Medium |
| P0-2 | Add `<entity>_sk` (MD5) and `<entity>_bk` to all 18 entities in Erwin model + DDL + pipeline code | SLV-001/002 | Data Modeling + ETL | Large |
| P0-3 | Change all monetary amount columns from STRING to DECIMAL(18,4); all date columns to DATE; all timestamp columns to TIMESTAMP UTC; all boolean flags to BOOLEAN | SLV-009/010, DQ Gate 2 | Data Modeling + ETL + DBA | Large |
| P0-4 | Remove 14 misscoped entities from `Txn - Electronic Channel Txn` SA in Erwin model (Customer, Event, Account Event variants, Financial Event variants, Reference_Business_SIC_History) | SA isolation | Data Modeling | Medium |
| P0-5 | Fix `Creditor Customer Number` and `Debtor Customer Number` mappings in Payment Platform Transaction — currently both map to `PRIME.CISOTRXNS.I042_MERCH_ID` (Merchant ID) | Semantic correctness | ETL / Source SME | Medium |
| P0-6 | Implement PhoneBanking sub-channel mapping: identify source system; complete 16 unmapped attributes | BCBS 239 completeness | ETL / Source SME | Large |

#### P1 — Next Sprint

| ID | Action | Rule | Owner | Effort |
|---|---|---|---|---|
| P1-1 | Migrate all `reference_*` tables to `silver.reference`: `reference_mcc`, `reference_merchant`, `reference_bank_branch`, `reference_currency`, `reference_transaction_code`, `reference_channel*` | SA isolation, SLV-004 | Data Modeling + ETL | Medium |
| P1-2 | Rename SCD columns: `Business Start Date` → `effective_from TIMESTAMP`, `Business End Date` → `effective_to TIMESTAMP`, `Is Active Flag` → `is_current BOOLEAN` | SLV-003 | Data Modeling + ETL + DBA | Medium |
| P1-3 | Add missing metadata columns to all entities: `record_hash`, `run_id`/`batch_id`, `source_ingestion_date`, `data_quality_status`, `dq_issues` | SLV-008 | Data Modeling + DBA | Medium |
| P1-4 | Apply SHA-256 masking to all card number fields in ECT and NECT-ATM pipeline transforms | PII Gate 2 | ETL | Small |
| P1-5 | Implement canonical code mappings for `transaction_type`, `transaction_status`, `segment`, `channel_type` via `silver.reference.code_mapping` | SLV-004 | ETL | Medium |
| P1-6 | Populate empty tables: `reference_mcc` (electronic schema), `reference_channel_type`, `reference_phonebankingagent` | Gate 2 Completeness | ETL | Small-Medium |
| P1-7 | Rename tables: `non_electronic_channel_transaction__pos_` → `non_electronic_channel_transaction_pos` · `non_electronic_channel_transaction__branch_` → `non_electronic_channel_transaction_branch` | NAM-001 | DBA | Small |
| P1-8 | Remove `balance_after_transaction` from `non_electronic_channel_transaction`; remove denormalized Party name/segment attributes from `payment_platform_transaction` and `internet_mobile_banking_transaction` | SLV-006/007 | Data Modeling + ETL | Small |
| P1-9 | Upgrade `payment_platform_transaction` Delta table properties: add ZSTD compression, autoOptimize, run OPTIMIZE | Platform standard | DBA / Platform | Small |

#### P2 — Backlog

| ID | Action | Rule | Owner | Effort |
|---|---|---|---|---|
| P2-1 | Design and implement missing double-entry posting entity (`transaction_posting` with debit/credit indicator and GL posting key) | BIAN/IFW Financial Transaction pattern | Data Modeling | Large |
| P2-2 | Fix triplicated SCD columns in `Non Electronic Channel Transaction (POS)` Erwin model | SLV-003 | Data Modeling | Small |
| P2-3 | Implement `Reference_ATM` — source from `silver.org.atm` per Organization SA; add FK from `non_electronic_channel_transaction_atm` | Referential integrity | Data Modeling + Org SA team | Medium |
| P2-4 | Map remaining 26 unmapped attributes in `Electronic Channel Transaction` | BCBS 239 completeness | ETL / Source SME | Large |
| P2-5 | Resolve `Reference_Transaction_Code` mapping ambiguity (`transaction_amount_limit` / `local` / `global` all from same source) | Semantic correctness | ETL / Source SME | Small |
| P2-6 | Address `Account_Event1` / `Account Event` duplication in Erwin model | Erwin governance | Data Modeling | Small |
| P2-7 | Register all Transaction SA tables in Informatica GDGC catalog with steward, classification, lineage | §2.12 Catalog registration | Data Governance | Medium |

### 7.2 Recommended Delivery Schedule

| Sprint | P0 Actions | P1 Actions |
|---|---|---|
| Sprint 1 (now) | P0-2 (SK/BK), P0-3 (types), P0-5 (PPT mapping fix) | P1-4 (PII), P1-7 (table rename) |
| Sprint 2 | P0-1 (schema rename), P0-4 (misscoped entities) | P1-2 (SCD columns), P1-8 (remove derived attrs) |
| Sprint 3 | P0-6 (PhoneBanking) | P1-1 (migrate reference data), P1-3 (metadata cols), P1-5 (code mapping), P1-6 (populate empty tables) |
| Sprint 4 | — | P1-9 (Delta upgrade) |
| Backlog | P2-1 through P2-7 | — |

---

## 8. Appendix

### Appendix A: Mapping Completeness Summary

| Entity | Total Attributes | Mapped | Not Available | % Mapped | Risk |
|---|---|---|---|---|---|
| Electronic Channel Transaction | 50 | 24 | 26 | 48% | High |
| Internet/Mobile Banking Transaction | 44 | 44 | 0 | 100% | Low |
| Payment Platform Transaction | 44 | 34 | 10 | 77% | Medium |
| Non Electronic Channel Transaction | 23 | 18 | 5 | 78% | Medium |
| Non Electronic Channel Transaction (Branch) | 17 | 6 | 11 | 35% | High |
| Non Electronic Channel Transaction (ATM) | 14 | 8 | 6 | 57% | High |
| Non Electronic Channel Transaction (POS) | 14 | 10 | 4 | 71% | Medium |
| Non Electronic Channel Transaction (PhoneBanking) | 17 | 1 | 16 | **6%** | Critical |
| Reference_Bank_Branch | 18 | 10 | 8 | 56% | Medium |
| Reference_Merchant | 28 | 21 | 7 | 75% | Medium |
| Reference_MCC | 13 | 9 | 4 | 69% | Medium |
| Reference_Transaction_Code | 24 | 20 | 4 | 83% | Low |
| Reference_Currency | 11 | 9 | 2 | 82% | Low |
| Reference_Channel | 14 | 0 | 14 | **0%** | High |
| Reference_Channel_Type | 10 | 0 | 10 | **0%** | High |
| Reference_Channel_Status | 8 | 0 | 8 | **0%** | Medium |
| Reference_Channel_Category_Type | 9 | 0 | 9 | **0%** | Medium |
| Reference_ATM | 14 | 0 | 14 | **0%** | High |
| Reference_PhoneBankingAgent | 11 | 3 | 8 | 27% | High |
| **TOTAL** | **393** | **217** | **176** | **55%** | |

---

### Appendix B: Guideline Citations

| Rule Code | Source Document | Exact Rule Text |
|---|---|---|
| SLV-001 | guidelines/11-modeling-dos-and-donts.md §2.4 | "Derive surrogate keys deterministically: `MD5(UPPER(TRIM(business_key)))` for single-component keys" |
| SLV-002 | guidelines/06-naming-conventions.md | "`<entity>_bk` — business/natural key" |
| SLV-003 | guidelines/11-modeling-dos-and-donts.md §2.3 | "Track all changes to tracked attributes with SCD Type 2: add a new row with updated values, close the prior row. Every Silver entity table must have `effective_from`, `effective_to`, `is_current`." |
| SLV-004 | guidelines/11-modeling-dos-and-donts.md §2.5 | "Resolve all source-system codes to canonical platform values using a central code-mapping reference (e.g., `silver.reference.code_mapping`). Store the canonical code, not the raw source code, in entity columns." |
| SLV-005 | guidelines/01-data-quality-controls.md §3.2 | "Failed records routed to quarantine / error tables rather than discarded. All DQ results must be logged and auditable." |
| SLV-006 | guidelines/11-modeling-dos-and-donts.md §2.6 | "Store only atomic, un-aggregated facts in Silver." |
| SLV-007 | guidelines/11-modeling-dos-and-donts.md §2.6 | "DON'T pre-compute totals, averages, counts, ratios, or any derived metrics in Silver tables." |
| SLV-008 | guidelines/11-modeling-dos-and-donts.md §2.7 | Mandatory Silver technical audit columns: `source_system_code`, `source_system_id`, `create_date`, `update_date`, `delete_date`, `is_active_flag`, `effective_from`, `effective_to`, `is_current`, `run_id`, `source_ingestion_date` |
| SLV-009 | guidelines/01-data-quality-controls.md §4 | "Time Zone Normalisation — All timestamps stored in UTC" · guidelines/11-modeling-dos-and-donts.md §2.9 |
| SLV-010 | guidelines/01-data-quality-controls.md §2, Gate 2 | "All monetary columns must use `DECIMAL(18,4)`" |
| NAM-001 | guidelines/06-naming-conventions.md | "Use snake_case for all identifiers. Use lowercase only. No abbreviations unless universally understood." |
| NAM-002 | guidelines/06-naming-conventions.md | "Silver: `silver.<subject_area>.<entity>`" |
| NAM-003 | guidelines/06-naming-conventions.md | "Silver entity: `<entity>`" |
| PII | guidelines/01-data-quality-controls.md §2, Gate 2 | "Card Numbers masked via SHA-256 unless stored in a restricted container" |

**Guideline Naming Conflict (for steward resolution):**
The two Silver-layer guidelines use different names for overlapping metadata columns:

| Concept | guidelines/01-data-quality-controls.md | guidelines/11-modeling-dos-and-donts.md |
|---|---|---|
| Source system identifier | `record_source` | `source_system_code` |
| SCD start | `valid_from_at` | `effective_from` |
| SCD end | `valid_to_at` | `effective_to` |
| Pipeline run ID | `batch_id` | `run_id` |
| Ingestion timestamp | `ingested_at` | `source_ingestion_date` |
| Processing timestamp | `processed_at` | `create_date` |

**Recommendation:** Adopt the modeling dos/donts (§2.7) names as canonical for Silver entity tables, and use DQ controls (§5) names for DQ tracking overlay columns (`record_hash`, `data_quality_status`, `dq_issues`). Both sets are needed; they serve complementary purposes. Data Governance Office should issue a unified Silver metadata standard to resolve the conflict definitively.

---

### Appendix C: Industry References

| Reference | Applicable Finding |
|---|---|
| **BIAN v11 — Service Domain: Financial Transaction** | ECT grain mismatch — BIAN defines `Financial Transaction` as a double-entry posting record, not a channel event record |
| **ISO 8583:2003** | PPT-002 — `I048_TEXT_DATA` is Additional Data field, not transaction status; `I042_MERCH_ID` is merchant ID not customer number (Finding PPT-001) |
| **ISO 20022 (pacs.008)** | `Payment Platform Transaction` should align with pacs.008 schema: `CdtTrfTxInf.Amt.InstdAmt`, `Dbtr.Id`, `Cdtr.Id` as distinct structured types |
| **BCBS 239 Principles 2, 3** | All-STRING types violate Principle 2 (Accuracy); 55% mapping completeness violates Principle 3 (Completeness) |
| **UAE CBUAE Circular 33/2021 (Payments)** | `channel_cross_border` unmapped; required for cross-border payment reporting; ATM transaction volume unreportable without `reference_atm` |
| **FATF Recommendation 16** | Phone Banking entity ~0% mapped; agent identity verification fields (`verified_flag`, `id_type`, `id`) unmapped — Wire Transfer Rule gap |
| **UAE PDPL Article 5** | Card number PII unmasked in Silver (ECT-004, ATM-001) |
| **IFW (IBM Industry Framework for Wholesale Banking)** | The double-entry posting pattern (`Debit_Financial_Transaction`/`Credit_Financial_Transaction` with `amount_signed BIGINT` and `dr_cr_indicator`) is the IFW standard for the Financial Transaction subject area |

---

*Report generated: 2026-04-27T14:22:11Z*  
*Assessment confidence: 0.92 (limited by absence of DDL — column-level physical analysis based on data mapping workbook)*  
*Next review date: 2026-07-27 (quarterly)*
