# Data Quality Controls
**Source:** Data Platform Development Standards and Naming Conventions V1.0 Draft  
**Owner:** Group Data Office  
**Version:** 1.0 Draft (December 2025)

---

## 1. Overview

Data quality is enforced at every transition boundary in the Medallion Architecture. Controls exist at three gates — Source → Bronze, Bronze → Silver, and Silver → Gold — and are supplemented by mandatory audit columns, DQ metrics, and platform-level KPIs.

---

## 2. Quality Gates

### Gate 1: Source → Bronze

| Aspect | Detail |
|--------|--------|
| **Checks** | File integrity; Schema Validation |
| **On Failure** | Failed files moved to `_error` folder |
| **On Success** | Files promoted to the Bronze layer |

### Gate 2: Bronze → Silver

| Check | Rule |
|-------|------|
| **Deduplication** | Remove duplicates based on Business Keys |
| **Completeness** | Silver must contain all required records for core entities (e.g., account tables include all types — no filtration on base attributes) |
| **Standardisation — Dates** | All date columns must conform to `YYYY-MM-DD` format |
| **Standardisation — Currencies** | All monetary columns must use `DECIMAL(18,4)` |
| **PII Masking** | EID, Card Numbers, Phone Numbers, and Passport Numbers masked via SHA-256 unless stored in a restricted container |
| **Error-Rate Threshold** | If the error rate exceeds **5%**, the pipeline must **FAIL** and alert the Engineering team |

**Action on failure:** Failed records are routed to a quarantine / Dead-Letter zone for investigation.

### Gate 3: Silver → Gold

| Check | Rule |
|-------|------|
| **Referential Integrity** | Every entity must resolve to a valid parent (e.g., a loan must have a valid `customer_id` in `dim_customer`) |
| **MDM Matching** | Entities matched against the Enterprise MDM Registry to retrieve a `Global_UUID`; source IDs are treated as secondary keys |

---

## 3. Silver Layer Data Quality & Validation Standards (§ 6.2.3)

### 3.1 Mandatory DQ Checks

| Dimension | Rule |
|-----------|------|
| **Completeness** | NOT NULL constraint enforced on all primary keys |
| **Validity** | Date ranges and numeric bounds validated against agreed domain rules |
| **Uniqueness** | Business key uniqueness enforced per table |
| **Referential Integrity** | All foreign keys must reference valid dimension records |
| **Accuracy** | Domain / value-set validation applied (e.g., enumerated codes, country lists) |

### 3.2 Enforcement

- Failed records routed to **quarantine / error tables** rather than discarded.
- All DQ results must be **logged and auditable**.

### 3.3 Minimum DQ Metrics

| Metric | Description |
|--------|-------------|
| **% Rejected Records** | Percentage of records failing any DQ rule per pipeline run |
| **% Nulls in Critical Fields** | Null rate for columns classified as mandatory or primary-key fields |
| **Duplicate Rate** | Percentage of duplicate records detected on business keys |
| **Freshness / Latency** | Time elapsed between source extraction and Silver availability |

---

## 4. Data Transformation Standards Relevant to Quality (§ 6.2.4)

| Rule | Requirement |
|------|-------------|
| **Deduplication** | Applied using business keys combined with timestamp |
| **Type Casting** | Explicit type casting to standard formats; no implicit conversion |
| **Time Zone Normalisation** | All timestamps stored in **UTC** |
| **Standard Code Mappings** | Country, currency, and status codes mapped to agreed reference values |
| **Aggregations / KPIs** | Prohibited in Silver; reserved for the Gold layer |

---

## 5. Mandatory Audit Columns (Quality Tracking)

Every Silver and Gold table must carry the following columns to support data quality traceability:

| Column | Type | Description |
|--------|------|-------------|
| `record_source` | STRING | Source system identifier |
| `record_origin` | STRING | Original schema / table name |
| `record_hash` | STRING | Hash used for deduplication |
| `is_current` | BOOLEAN | Indicates the current (active) row |
| `valid_from_at` | TIMESTAMP | Row validity start (SCD2) |
| `valid_to_at` | TIMESTAMP | Row validity end (SCD2) |
| `ingested_at` | TIMESTAMP | Timestamp of Bronze ingestion |
| `processed_at` | TIMESTAMP | Timestamp of Silver processing |
| `batch_id` | STRING | Pipeline run ID for lineage |
| `data_quality_status` | STRING | Outcome: `passed` or `quarantined` |
| `dq_issues` | STRING | JSON summary of individual rule failures |

---

## 6. Gold Layer Quality Standards (§ 6.3.4)

- **CI/CD Integration:** Schema tests, transformation tests, and integration tests are mandatory parts of the CI/CD pipeline.
- **Automated DQ Checks at Gold Refresh:** Every Gold promotion run must execute DQ checks before data is made available to consumers.
- **Alerts and Quarantine on Failure:** Failures trigger alerts; offending records are quarantined and do not reach the consumer-facing layer.

### 6.1 Metadata & Data Dictionary (§ 6.3.5)

No table may be promoted to the Gold layer without a **business-friendly description** registered in the data catalogue (CDGC).

**The description must answer:**
1. *What is it?*
2. *How is it calculated?*

| Quality | Example |
|---------|---------|
| ❌ Bad | "Customer Balance." |
| ✅ Good | "Total End-of-Day Ledger Balance in AED, excluding uncleared checks." |

---

## 7. KPI Framework — Data Quality Metrics (§ 9)

| Category | KPI Name | Definition | Target | Automated By |
|----------|----------|------------|--------|--------------|
| Governance | **DQI (Data Quality Index)** | % of rows passing all validation rules in the Gold layer | > 98% | ADF Data Flows |
| Governance | **CDE Lineage** | % of Regulatory fields (Basel III) with full lineage documented | 100% | Microsoft Purview |

---

## 8. Confluent Kafka Dead Letter Queue Standard (§ 5.2)

| Control | Requirement |
|---------|-------------|
| **Dead Letter Queue (DLQ)** | Every Source / Sink connector must route bad messages to `dlq.[topic_name]` |
| **Pipeline Stability** | Pipelines must **not** crash on schema mismatches; bad messages must be redirected |

---

## 9. ADF Pipeline Error Handling (§ 5.3)

| Control | Requirement |
|---------|-------------|
| **Retry Policy** | Exponential backoff with 3 retries: 30 s → 60 s → 120 s |
| **Error Routing** | Failed records redirected to a quarantine / Dead-Letter zone |
| **Alerting** | Critical pipelines must include alerting hooks (Azure Monitor / Log Analytics / custom dashboards) for near-real-time failure notification |
| **Idempotency** | Pipelines must be re-runnable without duplicating data (watermarks or CDC logic) |

---

## 10. PII & Sensitive Data Controls (§ 2.2)

| Control | Requirement |
|---------|-------------|
| **Consent Check** | All data entering the Gold layer must be joined with `dim_consent`; processing prohibited for opted-out customers (PDPL Article 5) |
| **Right to Be Forgotten** | Automated pipelines must support erasure of customer data within **30 days** |
| **Erasure Implementation** | Delta Lake `MERGE` to delete PI data, followed by `VACUUM (Retain 0)` to purge physical files |

---

*Document Version: 1.0 Draft — Aligned with Data Platform Development Standards and Naming Conventions V1.0 Draft*
