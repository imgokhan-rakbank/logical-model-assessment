# 11 — Data Modeling Gap Assessment: DOs and DON'Ts

## How to Use This Document

This document is an **LLM-ready gap assessment reference**. Feed it alongside your current logical and physical implementation (DDL, Erwin exports, pipeline code) to identify:

- 🔴 **Missing** — a required pattern or standard is entirely absent from the implementation.
- 🟡 **Improper** — the concept exists but is implemented incorrectly or sub-optimally.
- 🟢 **Proper** — the implementation matches the stated standard.

Each section covers one layer of the medallion architecture. Within each section, rules are stated as plain DO/DON'T pairs accompanied by concrete examples of both correct and incorrect implementation. Rules are intentionally generic and technology-neutral so they remain applicable regardless of whether the underlying engine is Databricks Delta Lake, Snowflake, BigQuery, or another platform.

---

## Summary: Layer Responsibilities at a Glance

| Layer | Primary Purpose | Modeling Approach | Consumers |
|-------|----------------|-------------------|-----------|
| **Bronze** | Immutable raw landing zone | Source-mirrored (no transformation) | Platform engineers, re-processing jobs |
| **Silver** | Single authoritative curated store | Third Normal Form (3NF) per subject area | Modelers, feature engineers, regulatory reporting |
| **Gold** | Purpose-built analytical consumption | Kimball star schema / aggregates | BI analysts, dashboards, data scientists |
| **Semantic** | Business-language metric abstraction | Views / metric definitions only (no storage) | Business users, self-service BI, external APIs |

Data flows in **one direction only**: Bronze → Silver → Gold → Semantic. No layer may read from a downstream layer.

---

## 1. Bronze Layer

### Purpose Recap
Bronze preserves every record exactly as received from source systems. It is an append-only, immutable audit trail. No business logic, filtering, renaming, or type conversion is permitted.

---

### 1.1 Immutability and Append-Only Writes

| | Rule |
|--|------|
| ✅ DO | Insert every inbound record. Use `INSERT` / `APPEND` mode only for CDC tables. |
| ❌ DON'T | Issue `UPDATE`, `DELETE`, or `TRUNCATE` / `OVERWRITE` on live CDC Bronze tables. |

**Improper — overwrites destroy history:**
```sql
-- ❌ WRONG: destroys all prior records
TRUNCATE TABLE bronze.cbs.accounts;
INSERT INTO bronze.cbs.accounts SELECT * FROM staging.accounts;
```

**Proper — append only:**
```sql
-- ✅ CORRECT: new records are appended, existing records preserved
INSERT INTO bronze.cbs.accounts
SELECT * FROM staging.accounts
WHERE _ingested_date = current_date();
```

> **Gap indicator:** If your Bronze tables have `UPDATE` or `DELETE` statements in any pipeline, that is a critical gap.

---

### 1.2 Source Fidelity — No Transformation at Bronze

| | Rule |
|--|------|
| ✅ DO | Copy source columns verbatim: same names, same data types, same values. |
| ❌ DON'T | Rename columns, cast types, apply filters, or add derived columns at the Bronze stage. |

**Improper — type cast and rename at ingestion:**
```sql
-- ❌ WRONG: changes the column name and casts the type
SELECT
    CAST(acct_bal AS DECIMAL(18,2)) AS account_balance,  -- renamed + cast
    UPPER(cust_nm)                  AS customer_name      -- transformed
FROM source.accounts;
```

**Proper — verbatim copy with platform metadata added separately:**
```sql
-- ✅ CORRECT: source columns unchanged; only platform metadata columns added
SELECT
    acct_bal,          -- as-is from source
    cust_nm,           -- as-is from source
    'CBS_FINACLE'      AS _source_system,
    current_timestamp() AS _ingested_at
FROM source.accounts;
```

> **Gap indicator:** If column names in Bronze differ from column names in the source system schema, that is a gap.

---

### 1.3 Mandatory Platform Metadata Columns

| | Rule |
|--|------|
| ✅ DO | Include all required platform metadata columns on every Bronze table: `_source_system`, `_source_table`, `_ingested_at`, `_ingested_date`, `_batch_id`, `_cdc_op`, `_cdc_timestamp`, `_schema_version`, `_checksum`, `_is_duplicate`. |
| ❌ DON'T | Omit any metadata column, or use inconsistent names across tables (e.g., `ingest_time` on one table and `_ingested_at` on another). |

**Improper — inconsistent and incomplete metadata:**
```sql
-- ❌ WRONG: missing columns, non-standard names
CREATE TABLE bronze.crm.customers (
    id          INT,
    name        STRING,
    load_date   TIMESTAMP   -- non-standard name; no _cdc_op, no _checksum, etc.
);
```

**Proper — complete standardised metadata:**
```sql
-- ✅ CORRECT: all 10 metadata columns present with standard names
CREATE TABLE bronze.crm.customers (
    id                  INT,
    name                STRING,
    _source_system      VARCHAR(64)     NOT NULL,
    _source_table       VARCHAR(128)    NOT NULL,
    _ingested_at        TIMESTAMP       NOT NULL,
    _ingested_date      DATE            NOT NULL,
    _batch_id           VARCHAR(64)     NOT NULL,
    _cdc_op             CHAR(1)         NOT NULL,
    _cdc_timestamp      TIMESTAMP,
    _schema_version     VARCHAR(16)     NOT NULL,
    _checksum           VARCHAR(64)     NOT NULL,
    _is_duplicate       BOOLEAN         NOT NULL
);
```

> **Gap indicator:** Any Bronze table missing one or more of the listed columns, or using non-standard column names, is a gap.

---

### 1.4 Partitioning

| | Rule |
|--|------|
| ✅ DO | Partition all Bronze tables by ingestion date (`_ingested_date`). |
| ❌ DON'T | Leave tables unpartitioned, or partition by a business date that may arrive out of order. |

**Improper — no partitioning:**
```sql
-- ❌ WRONG: full table scans for every incremental read
CREATE TABLE bronze.cbs.transactions (col1 STRING, col2 STRING);
```

**Proper — partitioned by ingestion date:**
```sql
-- ✅ CORRECT
CREATE TABLE bronze.cbs.transactions (col1 STRING, col2 STRING, _ingested_date DATE)
PARTITIONED BY (_ingested_date);
```

---

### 1.5 Schema Registration Before First Write

| | Rule |
|--|------|
| ✅ DO | Register the table schema in the catalog (e.g., Unity Catalog) before the first write. |
| ❌ DON'T | Let the pipeline auto-create the schema on first write without a reviewed DDL. |

> **Gap indicator:** Tables that exist in the platform but have no corresponding reviewed DDL or catalog registration entry.

---

### 1.6 CDC Dual-Table Pattern

| | Rule |
|--|------|
| ✅ DO | For CDC sources, maintain two tables: a near-real-time `_dlt`-suffixed table (continuous append) and an EOD-aligned table (no suffix, daily batch consolidation). |
| ❌ DON'T | Use the raw real-time CDC stream as the primary source for Silver pipelines in steady state. |

**Improper — Silver reads directly from the raw CDC stream:**
```python
# ❌ WRONG: Silver always reading from the noisy real-time CDC table
silver_input = spark.readStream.table("bronze.cbs.accounts_dlt")
```

**Proper — Silver reads EOD-aligned table (default) or _dlt only when low-latency is required:**
```python
# ✅ CORRECT: Silver reads the stable EOD-aligned snapshot
silver_input = spark.read.table("bronze.cbs.accounts").filter(f"_ingested_date = '{batch_date}'")
```

---

### 1.7 Data Quality at Bronze — Structural Checks Only

| | Rule |
|--|------|
| ✅ DO | Run structural DQ checks: metadata column completeness, row count volume alerting (e.g., >20% deviation from rolling average), schema version match, checksum uniqueness within a batch. |
| ❌ DON'T | Apply business-rule validation (domain checks, referential integrity) at Bronze. |

> **Gap indicator:** If business rules (e.g., "account_status must be ACTIVE or CLOSED") appear in Bronze pipeline code rather than Silver, that is a gap.

---

### 1.8 Access Control

| | Rule |
|--|------|
| ✅ DO | Restrict Bronze access to platform engineers and Silver pipeline service accounts only. |
| ❌ DON'T | Grant analysts, BI tools, or data scientists direct access to Bronze data. |

> **Gap indicator:** If any analyst role or BI service account has SELECT grants on Bronze catalogs/schemas, that is a gap.

---

## 2. Silver Layer

### Purpose Recap
Silver is the single source of truth for curated, canonicalized, DQ-validated data. It is organized into subject areas, modeled in Third Normal Form (3NF), and maintains full change history via SCD Type 2.

---

### 2.1 Third Normal Form (3NF) — No Repeating Groups, No Transitive Dependencies

| | Rule |
|--|------|
| ✅ DO | Decompose entities into 3NF: every non-key attribute depends on the whole primary key and nothing but the primary key. Repeating groups (sets of similar attributes that recur) must become child tables. |
| ❌ DON'T | Add numbered or suffixed columns to represent multiple instances of the same concept (e.g., `address1`, `address2`, `address3`, `address4`; `phone1`, `phone2`; `email_1`, `email_2`). |

**Guidance — mirror source multiplicity, avoid data loss**

When a source system expresses attributes as a 1-to-many set (for example, a customer may have many addresses or phone numbers), Silver should preserve that multiplicity. Avoid silently mapping a variable-length list into a fixed set of numbered columns — doing so is lossy and will drop items when the source emits more elements than the fixed columns allow. Preferred options:

- Normalize into a child table (one row per item) to preserve all items and support typed semantic fields and SCD semantics.
- When consumer convenience or performance justifies flattening, use a hybrid approach: keep the most-used attributes as flat columns and store the remaining items in a typed semi-structured column (e.g., `ARRAY<STRUCT<...>>` or JSON). This preserves all source data while providing easy access to the common fields.

#### Address Normalization Example (High-Signal Gap)

**Improper — numbered/suffixed address columns (repeating group, violates 1NF/3NF):**
```sql
-- ❌ WRONG: Address1–4 are a repeating group of free-form address lines.
-- Problems:
--   • Cannot distinguish address type (home vs. mailing vs. work)
--   • Cannot query "all customers in Dubai" without LIKE on 4 columns
--   • Cannot add a 5th address without a DDL change
--   • All 4 columns may be NULL except Address1, wasting space
--   • No structured fields (city, emirate, postal_code) → no analytics
CREATE TABLE silver.customer.contact (
    customer_sk   STRING NOT NULL,
    address1      STRING,
    address2      STRING,
    address3      STRING,
    address4      STRING
);
```
```

**Proper — separate normalized address entity with semantic fields:**
```sql
-- ✅ CORRECT: one row per address; typed semantic fields; supports multiple addresses per customer
CREATE TABLE silver.customer.customer_address (
    address_sk          STRING      NOT NULL,   -- MD5 surrogate
    customer_sk         STRING      NOT NULL,   -- FK to silver.customer.customer
    address_type_code   STRING      NOT NULL,   -- HOME / MAILING / WORK / PREVIOUS (canonical code)
    address_line_1      STRING      NOT NULL,   -- building / flat / street
    address_line_2      STRING,                 -- optional second line
    district            STRING,
    city                STRING      NOT NULL,
    emirate_code        STRING,                 -- UAE-specific: DXB / AUH / SHJ etc.
    postal_code         STRING,
    country_code        CHAR(3)     NOT NULL,   -- ISO 3166-1 alpha-3
    is_primary          BOOLEAN     NOT NULL,
    _valid_from         TIMESTAMP   NOT NULL,
    _valid_to           TIMESTAMP   NOT NULL,
    _is_current         BOOLEAN     NOT NULL,
    _source_system      STRING      NOT NULL,
    _silver_loaded_at   TIMESTAMP   NOT NULL,
    _silver_updated_at  TIMESTAMP   NOT NULL,
    _dq_status          STRING      NOT NULL,
    _dq_flags           ARRAY<STRING>,
    source_system_code     STRING,
    source_system_id       STRING,
    create_date            TIMESTAMP,
    update_date            TIMESTAMP,
    delete_date            TIMESTAMP,
    is_active_flag         STRING,
    effective_from         TIMESTAMP,
    effective_to           TIMESTAMP,
    is_current             BOOLEAN,
    run_id                 STRING,
    source_ingestion_date  TIMESTAMP,
    CONSTRAINT pk_customer_address PRIMARY KEY (address_sk)
);
```

Similarly apply this rule to phones, emails, identification documents:

```sql
-- ❌ WRONG: numbered phone columns
ALTER TABLE silver.customer.contact ADD COLUMN phone1 STRING;
ALTER TABLE silver.customer.contact ADD COLUMN phone2 STRING;

-- ✅ CORRECT: contact child table
CREATE TABLE silver.customer.customer_contact (
    contact_sk          STRING      NOT NULL,
    customer_sk         STRING      NOT NULL,
    contact_type_code   STRING      NOT NULL,   -- MOBILE / HOME / WORK / EMAIL
    contact_value       STRING      NOT NULL,
    is_primary          BOOLEAN     NOT NULL,
    is_verified         BOOLEAN     NOT NULL DEFAULT FALSE,
    verified_date       DATE,
    source_system_code     STRING,
    source_system_id       STRING,
    create_date            TIMESTAMP,
    update_date            TIMESTAMP,
    delete_date            TIMESTAMP,
    is_active_flag         STRING,
    effective_from         TIMESTAMP,
    effective_to           TIMESTAMP,
    is_current_flag        BOOLEAN,
    run_id                 STRING,
    source_ingestion_date  TIMESTAMP,
    CONSTRAINT pk_customer_contact PRIMARY KEY (contact_sk)
);
```

---

### 2.2 Subject-Area Isolation

| | Rule |
|--|------|
| ✅ DO | Keep each Silver subject area self-contained. Pipelines for one subject area read only from Bronze and from the same subject area's own Silver tables. |
| ❌ DON'T | Join across subject areas inside Silver pipelines (e.g., enriching `silver.account.account` with columns from `silver.customer.customer` in the same pipeline). |

**Improper — cross-subject-area join in a Silver pipeline:**
```sql
-- ❌ WRONG: account Silver pipeline joins customer Silver table
INSERT INTO silver.account.account
SELECT a.*, c.segment_code   -- cross-subject join
FROM bronze.cbs.accounts a
JOIN silver.customer.customer c ON a.customer_bk = c.customer_bk;
```

**Proper — account entity contains only account attributes; enrichment happens in Gold:**
```sql
-- ✅ CORRECT: Silver account entity contains only its own attributes
INSERT INTO silver.account.account
SELECT account_sk, account_bk, account_type_code, product_code,
       open_date, status_code, customer_bk, branch_code, ...
FROM bronze.cbs.accounts;
-- customer_sk join is done in gold.retail when building dim_account or fact tables
```

---

### 2.3 SCD Type 2 History

| | Rule |
|--|------|
| ✅ DO | Track all changes to tracked attributes with SCD Type 2: add a new row with updated values, close the prior row (`effective_to` = change timestamp, `is_current` = FALSE). Every Silver entity table must have `effective_from`, `effective_to`, `is_current`. |
| ❌ DON'T | Use SCD Type 1 (overwrite) for entities where historical state matters for regulatory, audit, or analytical purposes, without explicit data-steward approval. |

**Improper — in-place UPDATE losing history:**
```sql
-- ❌ WRONG: overwrites prior state; audit trail destroyed
UPDATE silver.customer.customer
SET segment_code = 'PREMIUM', updated_date = current_timestamp()
WHERE customer_bk = 'C-00123456';
```

**Proper — SCD-2 MERGE pattern:**
```sql
-- ✅ CORRECT: close old row, insert new row
MERGE INTO silver.customer.customer AS tgt
USING (SELECT 'C-00123456' AS customer_bk, 'PREMIUM' AS segment_code,
              current_timestamp() AS effective_from) AS src
ON tgt.customer_bk = src.customer_bk AND tgt.is_current = TRUE
WHEN MATCHED AND tgt.segment_code <> src.segment_code THEN
    UPDATE SET tgt.is_current = FALSE, tgt.effective_to = src.effective_from
WHEN NOT MATCHED THEN
    INSERT (customer_sk, customer_bk, segment_code, effective_from, effective_to, _is_current, ...)
    VALUES (...);
```

> **Gap indicator:** Silver tables that have `updated_date` but no `effective_from`, `effective_to`, or `is_current` columns are using SCD-1 (overwrite) by default — a potential gap depending on the entity.

---

### 2.4 Surrogate Keys — Deterministic, Not Auto-Increment

| | Rule |
|--|------|
| ✅ DO | Derive surrogate keys deterministically: `MD5(UPPER(TRIM(business_key)))` for single-component keys; `MD5(CONCAT_WS('|', key_part_1, key_part_2))` for composite keys. |
| ❌ DON'T | Use auto-increment sequences, `UUID()`, `NEWID()`, or `RAND()`-based surrogate keys. These break idempotency — a re-run or rebuild would generate different keys, severing downstream FK relationships in Gold. |

**Improper — non-deterministic surrogate:**
```sql
-- ❌ WRONG: different key assigned on every pipeline run
INSERT INTO silver.customer.customer (customer_sk, ...)
VALUES (UUID(), ...);
```

**Proper — deterministic MD5:**
```sql
-- ✅ CORRECT: same business key always produces the same surrogate
MD5(UPPER(TRIM(customer_bk)))  AS customer_sk
```

---

### 2.5 Business Key and Source Code Canonicalization

| | Rule |
|--|------|
| ✅ DO | Resolve all source-system codes to canonical platform values using a central code-mapping reference (e.g., `silver.reference.code_mapping`). Store the canonical code, not the raw source code, in entity columns. |
| ❌ DON'T | Store source-system-specific codes (e.g., CBS internal status codes like `01`, `AKT`, `9`) directly in Silver entity columns. |

**Improper — raw source code stored directly:**
```sql
-- ❌ WRONG: '01' is a CBS-internal code; meaningless to other consumers
INSERT INTO silver.account.account (account_status_code, ...)
VALUES ('01', ...);
```

**Proper — canonical code from mapping table:**
```sql
-- ✅ CORRECT: maps CBS '01' to canonical 'ACTIVE'
SELECT cm.canonical_code_value_bk AS account_status_code
FROM bronze.cbs.accounts a
JOIN silver.reference.code_mapping cm
  ON  cm.source_system = 'CBS_FINACLE'
  AND cm.source_domain = 'ACCOUNT_STATUS'
  AND cm.source_code   = a.acct_sts_cd
  AND cm.is_active     = TRUE;
```

> **Gap indicator:** Silver entity columns containing raw numeric or abbreviated source codes (e.g., `1`, `0`, `AKT`, `INA`) with no mapping table reference.

---

### 2.6 No Aggregations or Derived Metrics in Silver

| | Rule |
|--|------|
| ✅ DO | Store only atomic, un-aggregated facts in Silver. |
| ❌ DON'T | Pre-compute totals, averages, counts, ratios, or any derived metrics in Silver tables. |

**Improper — pre-aggregated metric in Silver:**
```sql
-- ❌ WRONG: total_transactions_30d is a derived metric; belongs in Gold
CREATE TABLE silver.customer.customer_profile (
    customer_sk             STRING,
    total_transactions_30d  INT,     -- aggregation; should not be in Silver
    avg_balance_90d         DECIMAL  -- aggregation; should not be in Silver
);
```
```

**Proper — atomic facts only; Gold computes aggregates:**
```sql
-- ✅ CORRECT: Silver stores individual transaction records
CREATE TABLE silver.financial_transaction.financial_transaction (
    transaction_sk      STRING  NOT NULL,
    account_sk          STRING  NOT NULL,
    transaction_date    DATE    NOT NULL,
    amount_aed_fils     BIGINT  NOT NULL,
    transaction_type_code STRING NOT NULL,
    -- No aggregations
    source_system_code     STRING,
    source_system_id       STRING,
    create_date            TIMESTAMP,
    update_date            TIMESTAMP,
    delete_date            TIMESTAMP,
    is_active_flag         STRING,
    effective_from         TIMESTAMP,
    effective_to           TIMESTAMP,
    is_current             BOOLEAN,
    run_id                 STRING,
    source_ingestion_date  TIMESTAMP
);
```

---

### 2.7 Mandatory Silver Technical Audit Columns

| | Rule |
|--|------|
| ✅ DO | Include the following Silver technical audit columns on every Silver entity table (names and types):
|
- `source_system_code` — `STRING`
- `source_system_id`   — `STRING`
- `create_date`        — `TIMESTAMP`
- `update_date`        — `TIMESTAMP`
- `delete_date`        — `TIMESTAMP` (nullable; used for soft-deletes)
- `is_active_flag`     — `STRING` (canonical active indicator: e.g., 'Y'/'N')
- `effective_from`     — `TIMESTAMP` (SCD-effective start)
- `effective_to`       — `TIMESTAMP` (SCD-effective end)
- `is_current`         — `BOOLEAN` (true for the current SCD row)
- `run_id`             — `STRING` (pipeline run identifier / batch id)
- `source_ingestion_date` — `TIMESTAMP` (when the source system emitted the record)

| ❌ DON'T | Omit lineage/audit columns or use inconsistent names across Silver tables (e.g., `load_ts` in one table and `create_date` in another). |

**Improper — inconsistent metadata and missing audit columns:**
```sql
-- ❌ WRONG: missing run id, inconsistent name for ingestion timestamp
CREATE TABLE silver.customer.customer (
    customer_sk      STRING NOT NULL,
    customer_bk      STRING NOT NULL,
    segment_code     STRING,
    _silver_updated_at TIMESTAMP  -- inconsistent with requested audit columns
);
```

**Proper — required technical audit columns present with canonical names:**
```sql
-- ✅ CORRECT: all mandatory technical audit columns present
CREATE TABLE silver.customer.customer (
    customer_sk            STRING    NOT NULL,
    customer_bk            STRING    NOT NULL,
    segment_code           STRING,
    source_system_code     STRING    NOT NULL,
    source_system_id       STRING,
    create_date            TIMESTAMP NOT NULL,
    update_date            TIMESTAMP,
    delete_date            TIMESTAMP,
    is_active_flag         STRING,
    effective_from         TIMESTAMP NOT NULL,
    effective_to           TIMESTAMP,
    is_current             BOOLEAN   NOT NULL,
    run_id                 STRING,
    source_ingestion_date  TIMESTAMP,
    CONSTRAINT pk_customer PRIMARY KEY (customer_sk)
);
```

**Gap indicator:** Any Silver table missing one or more of the listed technical audit columns or using non-standard/inconsistent column names is a gap (lineage, SCD, or auditability risk).

---

### 2.8 No SELECT * in Pipeline Code

| | Rule |
|--|------|
| ✅ DO | Always use explicit column lists in Silver (and Gold) SQL queries. |
| ❌ DON'T | Use `SELECT *` — it hides schema changes, can introduce unexpected columns, and makes column lineage impossible to trace. |

**Improper:**
```sql
-- ❌ WRONG
INSERT INTO silver.customer.customer SELECT * FROM staged_customers;
```

**Proper:**
```sql
-- ✅ CORRECT
INSERT INTO silver.customer.customer (customer_sk, customer_bk, segment_code, ...)
SELECT customer_sk, customer_bk, segment_code, ...
FROM staged_customers;
```

---

### 2.9 UTC Timestamps Only

| | Rule |
|--|------|
| ✅ DO | Store all timestamps in UTC. Convert source-local timestamps at ingestion. Where the source timezone is significant, store a companion `_source_tz` column. |
| ❌ DON'T | Store local-timezone timestamps (e.g., `Asia/Dubai` / GST+4) in primary timestamp columns. |

**Improper:**
```sql
-- ❌ WRONG: ambiguous timezone; daylight saving transitions cause duplicates
INSERT INTO silver.customer.customer (onboarding_date) VALUES (NOW());  -- system timezone
```

**Proper:**
```sql
-- ✅ CORRECT: explicit UTC conversion
CONVERT_TIMEZONE('Asia/Dubai', 'UTC', src.onboarding_ts)  AS onboarding_timestamp
```

---

### 2.10 Entity Modeling — Companion Tables for Multi-Source Attributes

| | Rule |
|--|------|
| ✅ DO | When the same logical entity is sourced from multiple systems (e.g., customer from CBS + CRM), create a canonical base entity table plus per-source companion tables. Merge or link them at Gold. |
| ❌ DON'T | Widen a single Silver table with source-prefixed columns (`cbs_email`, `crm_email`) or use nullable columns for attributes that only exist in some sources. |

**Improper — wide table with source-prefixed nullable columns:**
```sql
-- ❌ WRONG: sparse columns; source context lost
CREATE TABLE silver.customer.customer (
    customer_sk     STRING,
    cbs_segment     STRING,     -- NULL for CRM-sourced rows
    crm_segment     STRING,     -- NULL for CBS-sourced rows
    cbs_email       STRING,
    crm_email       STRING
);
```
```

**Proper — canonical entity + source-specific companion tables:**
```sql
-- ✅ CORRECT
CREATE TABLE silver.customer.customer (
    customer_sk     STRING NOT NULL,
    customer_bk     STRING NOT NULL,
    segment_code    STRING  -- canonical, resolved from code mapping
    -- only attributes that exist across all sources
);

CREATE TABLE silver.customer.customer (
    customer_sk     STRING NOT NULL,
    customer_bk     STRING NOT NULL,
    segment_code    STRING,
    source_system_code     STRING,
    source_system_id       STRING,
    create_date            TIMESTAMP,
    update_date            TIMESTAMP,
    delete_date            TIMESTAMP,
    is_active_flag         STRING,
    effective_from         TIMESTAMP,
    effective_to           TIMESTAMP,
    is_current             BOOLEAN,
    run_id                 STRING,
    source_ingestion_date  TIMESTAMP
);

CREATE TABLE silver.customer.customer_cbs_attributes (
    customer_sk     STRING NOT NULL,
    cbs_risk_band   STRING,
    cbs_product_group STRING
);

CREATE TABLE silver.customer.customer_cbs_attributes (
    customer_sk     STRING NOT NULL,
    cbs_risk_band   STRING,
    cbs_product_group STRING,
    source_system_code     STRING,
    source_system_id       STRING,
    create_date            TIMESTAMP,
    update_date            TIMESTAMP,
    delete_date            TIMESTAMP,
    is_active_flag         STRING,
    effective_from         TIMESTAMP,
    effective_to           TIMESTAMP,
    is_current             BOOLEAN,
    run_id                 STRING,
    source_ingestion_date  TIMESTAMP
);

CREATE TABLE silver.customer.customer_crm_attributes (
    customer_sk     STRING NOT NULL,
    crm_lead_source STRING,
    crm_lifecycle_stage STRING
);

CREATE TABLE silver.customer.customer_crm_attributes (
    customer_sk     STRING NOT NULL,
    crm_lead_source STRING,
    crm_lifecycle_stage STRING,
    source_system_code     STRING,
    source_system_id       STRING,
    create_date            TIMESTAMP,
    update_date            TIMESTAMP,
    delete_date            TIMESTAMP,
    is_active_flag         STRING,
    effective_from         TIMESTAMP,
    effective_to           TIMESTAMP,
    is_current             BOOLEAN,
    run_id                 STRING,
    source_ingestion_date  TIMESTAMP
);
```

---

### 2.11 Erwin Model Before DDL

| | Rule |
|--|------|
| ✅ DO | Create and approve the logical and physical model in Erwin (or equivalent modeling tool) before generating or writing DDL. DDL must be generated from the approved model. |
| ❌ DON'T | Hand-write DDL without a corresponding approved model. |

> **Gap indicator:** Tables that exist in the platform but have no corresponding entity in the Erwin model are a governance gap.

---

### 2.12 Informatica GDGC Registration

| | Rule |
|--|------|
| ✅ DO | Register every Silver table in the data catalog (Informatica GDGC or equivalent) with: data classification, data steward name, subject area, and lineage links. |
| ❌ DON'T | Deploy to production without catalog registration. |

> **Gap indicator:** Tables without a registered steward, classification tag, or lineage entry in the data catalog.

---

## 3. Gold Layer

### Purpose Recap
Gold contains purpose-built analytical models optimized for query performance. It follows Kimball dimensional modeling (star schema). Cross-subject-area joins and denormalization are performed here — never in Silver.

---

### 3.1 Star Schema — One Fact per Business Process

| | Rule |
|--|------|
| ✅ DO | Build one fact table per business process (account balance, loan disbursement, payment transaction, etc.). Each fact table is surrounded by conformed dimension tables connected via surrogate keys. |
| ❌ DON'T | Mix multiple business processes in a single fact table (e.g., account balances and loan repayments in one table). |

**Improper — mixed-process fact table:**
```sql
-- ❌ WRONG: account balance and loan repayment in the same fact table
CREATE TABLE gold.retail.fact_all_finance (
    account_sk          STRING,
    closing_balance     BIGINT,   -- account balance measure
    repayment_amount    BIGINT,   -- loan repayment measure
    interest_charged    BIGINT    -- lending measure
    -- grain is ambiguous; many nulls
);
```

**Proper — separate fact tables per process:**
```sql
-- ✅ CORRECT
CREATE TABLE gold.retail.fact_account_daily_balance (account_sk STRING, date_sk INT, closing_balance_aed_fils BIGINT, ...);
CREATE TABLE gold.risk.fact_loan_daily_position      (loan_sk STRING, date_sk INT, outstanding_principal_aed_fils BIGINT, ...);
```

---

### 3.2 Conformed Dimensions — Build Once, Reuse Everywhere

| | Rule |
|--|------|
| ✅ DO | Place shared dimensions (customer, date, product, branch) in a shared schema (e.g., `gold.shared`). Reference them by surrogate key from every fact table. |
| ❌ DON'T | Create duplicate dimension tables per domain (e.g., `gold.retail.dim_customer` AND `gold.risk.dim_customer` with different column sets). Duplicate dims cause conflicting attribute values across reports. |

**Improper — duplicate domain-specific dimensions:**
```sql
-- ❌ WRONG: same customer defined differently in two domains
CREATE TABLE gold.retail.dim_customer (customer_sk STRING, full_name STRING, segment_code STRING);
CREATE TABLE gold.risk.dim_customer   (customer_sk STRING, full_name STRING, risk_rating STRING);
-- Which one is correct? Joins to different tables give different attribute values.
```

**Proper — single conformed dimension in shared schema:**
```sql
-- ✅ CORRECT: one definition; both retail and risk fact tables point here
CREATE TABLE gold.shared.dim_customer (
    customer_sk     STRING NOT NULL,
    full_name       STRING,         -- access-controlled column
    segment_code    STRING,
    risk_rating     STRING,
    nationality_code CHAR(3),
    ...
);
```

---

### 3.3 No Snowflaking in Gold Dimensions

| | Rule |
|--|------|
| ✅ DO | Keep Gold dimensions fully denormalized (flat). Flatten all descriptive attributes into the dimension row, even if it means some redundancy. |
| ❌ DON'T | Normalize dimension attributes into sub-tables (snowflaking) — this defeats the query-performance purpose of Gold and forces additional joins in BI tools. |

**Improper — snowflaked product dimension:**
```sql
-- ❌ WRONG: BI tools must join dim_product → dim_product_category → dim_product_line
CREATE TABLE gold.shared.dim_product          (product_sk STRING, product_name STRING, category_sk STRING);
CREATE TABLE gold.shared.dim_product_category (category_sk STRING, category_name STRING, line_sk STRING);
CREATE TABLE gold.shared.dim_product_line     (line_sk STRING, line_name STRING);
```

**Proper — flat denormalized dimension:**
```sql
-- ✅ CORRECT: all hierarchy levels in one row
CREATE TABLE gold.shared.dim_product (
    product_sk          STRING NOT NULL,
    product_name        STRING,
    category_code       STRING,
    category_name       STRING,
    product_line_code   STRING,
    product_line_name   STRING
);
```

---

### 3.4 Surrogate Keys in Fact Tables (Not Business Keys)

| | Rule |
|--|------|
| ✅ DO | Use surrogate keys (SKs) for all FK relationships in fact tables. Surrogate keys support SCD-2 history in dimensions without breaking fact-to-dimension joins. |
| ❌ DON'T | Use business keys as FK columns in fact tables. If a customer's business key changes or a new SCD-2 row is added, business-key joins will break or produce incorrect results. |

**Improper — business key FK in fact:**
```sql
-- ❌ WRONG: joining fact to dim by business key returns all SCD-2 rows, not just the one current at transaction time
CREATE TABLE gold.retail.fact_transaction (
    transaction_sk  STRING,
    customer_bk     STRING,  -- BK FK; ambiguous across SCD-2 history
    amount          BIGINT
);
```

**Proper — surrogate key FK:**
```sql
-- ✅ CORRECT: SK uniquely identifies a specific point-in-time dimension row
CREATE TABLE gold.retail.fact_transaction (
    transaction_sk  STRING NOT NULL,
    customer_sk     STRING NOT NULL,  -- SK FK → gold.shared.dim_customer
    date_sk         INT    NOT NULL,  -- SK FK → gold.shared.dim_date
    amount_aed_fils BIGINT NOT NULL
);
```

---

### 3.5 Gold Sources — Silver or Gold Only (Never Bronze)

| | Rule |
|--|------|
| ✅ DO | Gold pipelines read from Silver tables or from other Gold tables. |
| ❌ DON'T | Read from Bronze tables directly in Gold pipelines. Bronze data has not been DQ-validated, canonicalized, or SCD-2 managed. |

> **Gap indicator:** Any Gold pipeline with a `FROM bronze.*` reference is a critical gap.

---

### 3.6 Incremental Load with Watermark

| | Rule |
|--|------|
| ✅ DO | Use watermark-based incremental loads (`WHERE _silver_updated_at > last_gold_watermark`). Apply ACID MERGE (not INSERT-then-UPDATE) on the fact or dimension surrogate key. |
| ❌ DON'T | Rebuild entire Gold tables from scratch on every batch run in steady state. Full rebuilds are acceptable only for initial load or disaster recovery. |

**Improper — full daily rebuild:**
```sql
-- ❌ WRONG: scans entire Silver table daily regardless of what changed
TRUNCATE TABLE gold.retail.fact_account_daily_balance;
INSERT INTO gold.retail.fact_account_daily_balance SELECT ... FROM silver.account.account_balance;
```

**Proper — watermark-based MERGE:**
```sql
-- ✅ CORRECT: only changed records are processed
MERGE INTO gold.retail.fact_account_daily_balance tgt
USING (
    SELECT ... FROM silver.account.account_balance
    WHERE _silver_updated_at > (SELECT MAX(_silver_updated_at) FROM gold.retail.fact_account_daily_balance)
) src ON tgt.balance_sk = src.balance_sk
WHEN MATCHED THEN UPDATE SET *
WHEN NOT MATCHED THEN INSERT *;
```

---

### 3.7 Post-Load Optimization

| | Rule |
|--|------|
| ✅ DO | Run file compaction (`OPTIMIZE`) and statistics-based clustering (e.g., `ZORDER BY`) after each incremental load. Vacuum old file versions weekly. |
| ❌ DON'T | Leave Gold tables as thousands of small files; this degrades query performance and increases scan costs. |

---

### 3.8 Additive Measures in Fact Tables

| | Rule |
|--|------|
| ✅ DO | Store additive raw measures in fact tables (amounts, counts, durations). Non-additive metrics (ratios, percentages) are computed at query time in Semantic views or pre-computed in aggregation tables with explicit labeling. |
| ❌ DON'T | Store non-additive pre-computed ratios (e.g., `npl_ratio`, `cost_to_income_ratio`) as raw columns in a fact table — summing them across rows is mathematically incorrect. |

**Improper — non-additive ratio stored as a fact measure:**
```sql
-- ❌ WRONG: SUM(npl_ratio) across customer rows is meaningless
CREATE TABLE gold.risk.fact_loan_daily (
    loan_sk     STRING,
    npl_ratio   DECIMAL(10,4)  -- non-additive; can't be summed
);
```

**Proper — additive components stored; ratio computed at query time:**
```sql
-- ✅ CORRECT: store the components; compute ratio in Semantic view
CREATE TABLE gold.risk.fact_loan_daily (
    loan_sk                         STRING  NOT NULL,
    npl_outstanding_principal_fils  BIGINT, -- additive
    total_outstanding_principal_fils BIGINT  -- additive
);
-- Ratio computed in semantic.v_npl_ratio: SUM(npl) / SUM(total)
```

---

### 3.9 Referential Integrity Checks

| | Rule |
|--|------|
| ✅ DO | Validate after every load that every FK in a fact table resolves to a row in its dimension. Alert on orphaned fact rows. |
| ❌ DON'T | Load fact rows without confirming dimension membership — orphaned facts cause silent gaps in BI reports. |

---

### 3.10 Access Control

| | Rule |
|--|------|
| ✅ DO | Apply column-level security on classified attributes (e.g., `nationality_code`, `full_name`). Apply row-level security for multi-tenant domains (e.g., branch-scoped access). |
| ❌ DON'T | Grant unrestricted SELECT on all Gold columns to all analyst roles. |

---

## 4. Semantic Layer

### Purpose Recap
The Semantic layer is the certified business-language interface. It contains no physical storage — only SQL views and metric definitions over Gold tables. Every consumer-facing KPI has exactly one definition here.

---

### 4.1 No Physical Storage

| | Rule |
|--|------|
| ✅ DO | Implement the Semantic layer exclusively as SQL views (or metric-layer definitions in BI/semantic tools like dbt Metrics, Cube.dev, or Power BI datasets). |
| ❌ DON'T | Create physical tables in the Semantic layer. Physical storage belongs in Gold. |

> **Gap indicator:** If `CREATE TABLE` statements exist under the `semantic` catalog/schema, that is a structural gap.

---

### 4.2 Gold-Only Sources

| | Rule |
|--|------|
| ✅ DO | Semantic views reference only Gold tables. |
| ❌ DON'T | Reference Silver or Bronze tables directly from Semantic views. |

**Improper:**
```sql
-- ❌ WRONG: Semantic view bypasses Gold and reads Silver directly
CREATE VIEW semantic.retail.v_active_customers AS
SELECT COUNT(*) FROM silver.customer.customer WHERE _is_current = TRUE;
```

**Proper:**
```sql
-- ✅ CORRECT: reads from Gold
CREATE VIEW semantic.retail.v_active_customers AS
SELECT COUNT(DISTINCT customer_sk) FROM gold.shared.dim_customer
WHERE relationship_end_date IS NULL;
```

---

### 4.3 One Certified Definition per Metric

| | Rule |
|--|------|
| ✅ DO | Maintain exactly one SQL view (or metric definition) per KPI. Certify it through a documented workflow (Draft → Review → Business Approval → Certified). Record definition, formula, grain, and owner in the data catalog. |
| ❌ DON'T | Allow the same metric to be defined in multiple BI tools, reports, or views with different formulas. Conflicting definitions produce different numbers for the same KPI in different reports — a critical credibility failure. |

**Improper — duplicate metric definitions in different tools:**
```
# ❌ WRONG: three different definitions of "active customer count" exist
Power BI dataset:  COUNT(customer_id) WHERE status = 'A'
Tableau workbook:  COUNT(DISTINCT customer_sk) WHERE has_active_account = true
Semantic view:     COUNT(*) FROM dim_customer WHERE relationship_end_date IS NULL
# All three give different numbers → loss of trust in data
```

**Proper — single certified view:**
```sql
-- ✅ CORRECT: one certified definition; all BI tools connect to this view
CREATE OR REPLACE VIEW semantic.retail.v_active_retail_customer_count AS
SELECT
    d.full_date       AS reporting_date,
    COUNT(DISTINCT f.customer_sk) AS active_retail_customer_count
FROM gold.retail.fact_account_daily_balance f
JOIN gold.shared.dim_customer c ON f.customer_sk = c.customer_sk
JOIN gold.shared.dim_date     d ON f.date_sk      = d.date_sk
WHERE c.customer_type_code = 'RETAIL'
  AND f.account_status_code = 'ACT'
GROUP BY d.full_date;
-- Documented in catalog: owner = Retail Analytics, certified 2024-01-15
```

---

### 4.4 BI Tools Connect to Semantic Layer Only

| | Rule |
|--|------|
| ✅ DO | All BI tools (Power BI, Tableau, Superset, Databricks SQL, etc.) must connect exclusively to `semantic.*` views. |
| ❌ DON'T | Allow direct connections from BI tools to Gold, Silver, or Bronze catalogs. Direct Gold access bypasses metric certification and column-level security applied at the Semantic view layer. |

> **Gap indicator:** Power BI or Tableau data sources pointing to `gold.*` or `silver.*` objects instead of `semantic.*` views.

---

### 4.5 Metric Versioning on Breaking Changes

| | Rule |
|--|------|
| ✅ DO | When a metric formula changes in a way that alters its output (breaking change), create a new versioned view (e.g., `v_npl_ratio_v2`) and run both in parallel during a migration window. Deprecate the old version with a sunset date. |
| ❌ DON'T | Silently change an existing certified view's formula — this breaks all downstream reports that depend on the prior definition. |

---

### 4.6 Non-Additive Metrics Computed Here

| | Rule |
|--|------|
| ✅ DO | Compute ratios, percentages, and other non-additive metrics in Semantic views using the correct aggregation of the underlying additive Gold measures. |
| ❌ DON'T | Pre-compute ratios in Gold and store them as columns — they cannot be correctly summed or sliced by dimensions in BI tools. |

**Improper — ratio pre-computed in Gold fact:**
```sql
-- ❌ WRONG: ratio column stored in Gold; cannot be aggregated by BI tool
CREATE TABLE gold.risk.fact_loan_daily (
    loan_sk     STRING,
    npl_ratio   FLOAT   -- 0.023; cannot be summed across rows
);
```

**Proper — ratio derived in Semantic view from additive components:**
```sql
-- ✅ CORRECT
CREATE VIEW semantic.risk.v_npl_ratio AS
SELECT
    d.full_date,
    SUM(npl_outstanding_principal_fils)   AS npl_amount,
    SUM(total_outstanding_principal_fils) AS total_amount,
    ROUND(SUM(npl_outstanding_principal_fils) * 1.0
          / NULLIF(SUM(total_outstanding_principal_fils), 0), 6) AS npl_ratio
FROM gold.risk.fact_loan_daily l
JOIN gold.shared.dim_date d ON l.date_sk = d.date_sk
GROUP BY d.full_date;
```

---

## 5. Cross-Cutting Rules (Apply to All Layers)

### 5.1 Unidirectional Data Flow

| | Rule |
|--|------|
| ✅ DO | Enforce Bronze → Silver → Gold → Semantic data flow direction through access controls (catalog grants), not just convention. |
| ❌ DON'T | Allow any layer to read from a downstream layer (e.g., Silver reading from Gold, Gold reading from Semantic). |

> **Gap indicator:** Any SELECT grant on Gold or Semantic catalogs for a Silver pipeline service account.

---

### 5.2 Naming Conventions

| | Rule |
|--|------|
| ✅ DO | Use snake_case for all object names. Follow layer-specific prefixes/suffixes: fact tables prefixed `fact_`, dimension tables prefixed `dim_`, aggregation tables prefixed `agg_`, Semantic views prefixed `v_`. |
| ❌ DON'T | Mix naming styles (camelCase, PascalCase, ALL_CAPS) across tables. Avoid abbreviations that are not in the approved list. |

**Improper naming examples:**
```
bronze.CBS.AccountMaster       -- PascalCase, uppercase schema
silver.cust.Cust               -- cryptic abbreviation, mixed case
gold.retail.FactAcctDailyBal   -- PascalCase fact table
```

**Proper naming examples:**
```
bronze.cbs.account_master
silver.customer.customer
gold.retail.fact_account_daily_balance
```

---

### 5.3 No Business Logic in Bronze; No Raw Data in Gold

| | Rule |
|--|------|
| ✅ DO | Enforce the layered responsibility model: transformation is for Silver, aggregation is for Gold, abstraction is for Semantic. |
| ❌ DON'T | Apply business rules (domain validation, code mapping) in Bronze pipelines, or expose raw uncanonicalized data directly in Gold. |

---

### 5.4 Idempotent Pipelines

| | Rule |
|--|------|
| ✅ DO | Design every pipeline to be idempotent: running it twice with the same input produces the same output (no duplicates, no data loss). Use MERGE on deterministic keys, not INSERT-always. |
| ❌ DON'T | Use INSERT without duplicate checks. A re-run should not double-count records. |

---

### 5.5 No EAV (Entity-Attribute-Value) Tables

| | Rule |
|--|------|
| ✅ DO | Use typed relational columns for all known attributes. For genuinely dynamic schemas, use `MAP<STRING, STRING>` or `VARIANT`/`JSON` typed columns — not a generic key-value table. |
| ❌ DON'T | Model well-known business attributes as rows in an EAV table (e.g., `attribute_name VARCHAR, attribute_value VARCHAR`). EAV tables are unqueryable, have no type safety, and perform poorly at scale. |

**Improper — EAV for customer attributes:**
```sql
-- ❌ WRONG: EAV; impossible to enforce types or apply DQ rules
CREATE TABLE silver.customer.customer_attribute (
    customer_sk     STRING,
    attribute_name  STRING,   -- 'segment', 'kyc_status', 'nationality'
    attribute_value STRING    -- all values as strings; no type safety
);

```sql
-- ✅ CORRECTED: EAV should include audit columns if used (but prefer typed columns instead)
CREATE TABLE silver.customer.customer_attribute (
    customer_sk     STRING,
    attribute_name  STRING,
    attribute_value STRING,
    source_system_code     STRING,
    source_system_id       STRING,
    create_date            TIMESTAMP,
    update_date            TIMESTAMP,
    delete_date            TIMESTAMP,
    is_active_flag         STRING,
    effective_from         TIMESTAMP,
    effective_to           TIMESTAMP,
    is_current             BOOLEAN,
    run_id                 STRING,
    source_ingestion_date  TIMESTAMP
);
```
```

**Proper — typed columns:**
```sql
-- ✅ CORRECT: each attribute is a typed column with its own DQ rule
CREATE TABLE silver.customer.customer (
    customer_sk      STRING  NOT NULL,
    segment_code     STRING,
    kyc_status_code  STRING  NOT NULL,
    nationality_code CHAR(3)
);

```sql
-- ✅ CORRECT: each attribute is a typed column with its own DQ rule; include audit columns
CREATE TABLE silver.customer.customer (
    customer_sk      STRING  NOT NULL,
    segment_code     STRING,
    kyc_status_code  STRING  NOT NULL,
    nationality_code CHAR(3),
    source_system_code     STRING,
    source_system_id       STRING,
    create_date            TIMESTAMP,
    update_date            TIMESTAMP,
    delete_date            TIMESTAMP,
    is_active_flag         STRING,
    effective_from         TIMESTAMP,
    effective_to           TIMESTAMP,
    is_current             BOOLEAN,
    run_id                 STRING,
    source_ingestion_date  TIMESTAMP
);
```
```

---

### 5.6 Deeply Nested JSON in Analytical Tables

| | Rule |
|--|------|
| ✅ DO | Flatten JSON payloads to typed columns at Silver ingestion. Use `STRUCT` / nested types only when the schema is genuinely dynamic or the nesting is shallow and well-documented. |
| ❌ DON'T | Store deeply nested JSON blobs in Silver or Gold tables that analytical queries must navigate with `JSON_EXTRACT` paths. |

---

### 5.7 Deprecation Before Deletion

| | Rule |
|--|------|
| ✅ DO | Mark columns or tables as deprecated in the catalog with a sunset date (minimum 6-month grace period). Notify downstream consumers. Only physically drop after the grace period has passed. |
| ❌ DON'T | Drop columns or tables immediately — even if no known consumers are registered. |

---

## 6. Gap Assessment Quick-Reference Matrix

Use this matrix when comparing an actual implementation against these rules. For each rule, assess the implementation and assign one of: 🟢 Proper / 🟡 Improper / 🔴 Missing / ⚪ Not Applicable.

| # | Area | Rule Summary | Layer(s) |
|---|------|-------------|---------|
| 1 | Immutability | Bronze tables are append-only; no UPDATE/DELETE | Bronze |
| 2 | Source Fidelity | Column names and types match source exactly in Bronze | Bronze |
| 3 | Metadata Columns | All 10 Bronze metadata columns present on every table | Bronze |
| 4 | Partitioning | Bronze tables partitioned by `_ingested_date` | Bronze |
| 5 | CDC Dual-Table | Near-real-time `_dlt` table + EOD-aligned table per CDC source | Bronze |
| 6 | Bronze DQ | Only structural checks at Bronze (no business rules) | Bronze |
| 7 | Bronze Access | Analysts have no access to Bronze | Bronze |
| 8 | 3NF Normalization | No repeating groups; no transitive dependencies | Silver |
| 9 | Address/Phone/Email | Multi-value contact attributes in child tables (not numbered columns) | Silver |
| 10 | Subject-Area Isolation | No cross-subject-area joins inside Silver pipelines | Silver |
| 11 | SCD-2 History | `_valid_from`, `_valid_to`, `_is_current` on all Silver entity tables | Silver |
| 12 | Deterministic SKs | Surrogate keys are MD5(BK), not UUID/sequence | Silver |
| 13 | Code Canonicalization | Source codes mapped to canonical values via reference table | Silver |
| 14 | No Aggregations | Silver tables contain only atomic facts | Silver |
| 15 | Silver Metadata Cols | All 9 Silver metadata columns on every table | Silver |
| 16 | No SELECT * | Explicit column lists in all Silver SQL | Silver |
| 17 | UTC Timestamps | All Silver timestamps in UTC | Silver |
| 18 | Erwin Model | Logical/physical model in Erwin before DDL generated | Silver |
| 19 | Catalog Registration | Every Silver table registered in GDGC with steward + classification | Silver |
| 20 | Star Schema | Gold fact tables follow one-fact-per-business-process pattern | Gold |
| 21 | Conformed Dims | Shared dimensions in `gold.shared`; no per-domain duplicates | Gold |
| 22 | No Snowflaking | Gold dimensions are fully denormalized/flat | Gold |
| 23 | SK FKs in Facts | Fact FK columns use surrogate keys, not business keys | Gold |
| 24 | Gold Sources | Gold reads only Silver or Gold (never Bronze) | Gold |
| 25 | Incremental Load | Watermark-based MERGE; no full daily rebuilds | Gold |
| 26 | Post-Load Optimize | OPTIMIZE / ZORDER run after every load | Gold |
| 27 | Additive Measures | Non-additive ratios not stored as fact columns | Gold |
| 28 | RI Checks | FK-to-PK referential integrity validated post-load | Gold |
| 29 | No Physical Storage | Semantic layer contains views only (no tables) | Semantic |
| 30 | Gold-Only Sources | Semantic views read only Gold tables | Semantic |
| 31 | One Metric Definition | Each KPI has exactly one certified Semantic view | Semantic |
| 32 | BI Routing | All BI tools connect to `semantic.*` only | Semantic |
| 33 | Metric Versioning | Breaking formula changes produce a new versioned view | Semantic |
| 34 | Non-Additive Metrics | Ratios/percentages computed in Semantic, not stored in Gold | Semantic |
| 35 | Unidirectional Flow | No layer reads from a downstream layer | All |
| 36 | Naming Conventions | snake_case; approved prefixes/suffixes; no mixed case | All |
| 37 | Idempotent Pipelines | Re-running produces same result; MERGE on deterministic keys | All |
| 38 | No EAV Tables | Known attributes modeled as typed columns, not key-value rows | All |
| 39 | No Deeply Nested JSON | JSON flattened to typed columns at Silver; no analytic JSON blobs | All |
| 40 | Deprecation Grace Period | Columns/tables deprecated with 6-month notice before physical drop | All |