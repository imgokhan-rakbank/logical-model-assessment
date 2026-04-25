# Naming Conventions, Versioning and Migrations

## Naming Conventions *(NAM-001)*

### General Rules
- Use **snake_case** for all identifiers (catalogs, schemas, tables, columns, views). *(NAM-001)*
- Use **lowercase only** — no mixed case.
- No abbreviations unless they are universally understood domain terms (see approved list below).
- Maximum identifier length: **63 characters** (Databricks / most SQL engines).
- Never use SQL reserved words as identifiers (e.g., `date`, `value`, `order`). *(NAM-005)*

### Approved Abbreviations

| Abbreviation | Full Term |
|-------------|-----------|
| `id` | Identifier (only for natural/business IDs from source) |
| `sk` | Surrogate Key |
| `bk` | Business Key |
| `amt` | Amount |
| `ccy` | Currency |
| `dt` | Date |
| `ts` | Timestamp |
| `cd` | Code |
| `nm` | Name (use sparingly; prefer `name`) |
| `flg` | Flag |
| `cnt` | Count |
| `pct` | Percentage |
| `yr` | Year |
| `mo` | Month |
| `aed` | UAE Dirham (currency suffix) |

---

## Catalog / Schema / Database Naming *(NAM-002)*

The platform uses **Databricks Unity Catalog 3-level namespace**: `<catalog>.<schema>.<table>`. 

| Layer | Catalog | Schema | Full Pattern | Example |
|-------|---------|--------|--------------|---------|
| Bronze | `bronze` | `<source_system>` | `bronze.<source>.<source_schema>_<table>` | `bronze.finacle.crmuser_accounts` |
| Silver | `silver` | `<subject_area>` | `silver.<subject_area>.<entity>` | `silver.customer.customer`, `silver.financial_transaction.financial_transaction` |
| Gold | `gold` | `<domain>` | `gold.<domain>.<table>` | `gold.retail.fact_account_daily_balance` |
| Gold Shared Dims | `gold` | `shared` | `gold.shared.<dim_table>` | `gold.shared.dim_customer`, `gold.shared.dim_date` |
| Semantic | `semantic` | `<domain>` | `semantic.<domain>.<view>` | `semantic.retail.v_active_customers` |

---

## Table Naming *(NAM-003)*

```
<subject>_<entity>[_<qualifier>]
```

| Type | Pattern | Example |
|------|---------|---------|
| Fact table | `fact_<subject>_<grain>` | `fact_account_daily_balance`, `fact_loan_monthly` |
| Dimension | `dim_<entity>` | `dim_customer`, `dim_date`, `dim_product` |
| Bridge / associative | `brd_<entity_a>_<entity_b>` | `brd_customer_account` |
| Silver entity | `<entity>` | `customer`, `account`, `transaction` |
| Bronze raw | `<source_table_name>` | `accounts` (as-is from source, lowercased) |
| Aggregate | `agg_<subject>_<grain>` | `agg_customer_monthly`, `agg_branch_daily` |
| Stage | `stg_<entity>` | `stg_customer_clean` |
| Snapshot | `<entity>_snapshot` | `customer_snapshot` |

---

## Column Naming

### Primary Key and Foreign Key
```
<entity>_sk   -- surrogate key (Gold dimensions, Silver entities)
<entity>_bk   -- business/natural key
<entity>_id   -- source-system ID (Bronze only)
<entity>_sk   -- FK in fact tables (suffix matches dim table PK)
```

### Standard Business Columns
```sql
-- Monetary amounts
<measure>_amount_<currency>   -- e.g., closing_balance_amount_aed, interest_amount_usd
<measure>_amount_orig         -- original currency amount
currency_code                 -- ISO 4217 3-letter code

-- Date/time
<event>_date          -- DATE type:      account_open_date, payment_value_date
<event>_timestamp     -- TIMESTAMP type: transaction_timestamp, event_timestamp
effective_date        -- SCD-2 effective date (if relevant)
expiry_date           -- expiry/end date

-- Status and codes
<entity>_status_code  -- status_code is a discriminator: account_status_code
<entity>_type_code    -- type discriminator: product_type_code
<entity>_category_code

-- Flags (Boolean)
is_<condition>        -- is_active, is_current, is_deleted, is_npl
has_<condition>       -- has_collateral, has_insurance

-- Metadata (see layer-specific standards for full list)
_<platform_column>    -- all platform/metadata columns prefixed with underscore
```

---

## View and Object Naming

| Object Type | Pattern | Example |
|-------------|---------|---------|
| SQL View | `v_<entity_or_metric>` | `v_active_customers`, `v_npl_ratio` |
| Materialized View | `mv_<entity>_<grain>` | `mv_account_daily` |
| External Table | `ext_<source>_<entity>` | `ext_swift_messages` |
| Stored Procedure | `sp_<action>_<subject>` | `sp_load_customer` |
| Pipeline / Job | `<layer>_<subject>_<action>` | `silver_customer_scd2_merge` |

---

## Versioning Strategy

### Model Versioning
- Models are versioned when a **breaking change** is introduced (column removed, key changed, grain altered).
- Non-breaking changes (adding nullable columns, new metadata) do **not** require a version bump.
- Version suffix: `_v<N>` appended to the table/view name.

```
silver.customer.customer        -- current/active version
silver.customer.customer_v1     -- deprecated but retained for 6 months
```

### Schema Versioning (DDL)
- All DDL changes tracked as numbered migration scripts in Git:
  ```
  migrations/
  ├── silver.customer/
  │   ├── V001__create_customer.sql
  │   ├── V002__add_segment_code.sql
  │   └── V003__add_kyc_status.sql
  ```
- Migration tool: **Flyway** or **Liquibase** for RDBMS; **Delta DDL scripts** tracked manually for lakehouse.
- Migrations run in CI pipeline before model deployment.

---

## Schema Migration Rules

| Change Type | Classification | Migration Required | Backward Compatible |
|-------------|---------------|-------------------|---------------------|
| Add nullable column | Non-breaking | Version script | ✅ Yes |
| Add NOT NULL column with default | Non-breaking | Version script | ✅ Yes |
| Rename column | Breaking | New version + alias view | ❌ No |
| Drop column | Breaking | New table version | ❌ No |
| Change data type (widening) | Non-breaking | Version script | ✅ Yes |
| Change data type (narrowing) | Breaking | New table version | ❌ No |
| Change primary key | Breaking | New table version | ❌ No |
| Add partition column | Breaking | Full rebuild required | ❌ No |

**Breaking change procedure:**
1. Create new table `<name>_v<N+1>` with the new schema.
2. Migrate data from old table to new.
3. Update all downstream pipeline references to new table.
4. Deprecate old table — mark in catalog; set `_deprecated = TRUE` in table properties.
5. Retain old table for **6 months** before physical deletion.
6. Communicate deprecation via data catalog announcement and pipeline owner notification.

---