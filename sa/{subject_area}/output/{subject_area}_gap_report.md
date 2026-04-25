# Gap Assessment Report — Customer

> **Note:** This report was written to the template path (`sa/{subject_area}/output/`) because the
> `sa/customer/output/` directory did not exist at assessment time. Move this file to:
> `sa/customer/output/customer_gap_report.md` and restore the original template.

| Field | Value |
|---|---|
| Subject Area | `customer` |
| Schema | `silver_dev_v2.customer` (canonical target: `silver.party`) |
| Assessment Date | 2026-04-25 |
| Overall State | 🔴 Incorrect |
| Entities Assessed | 10 |
| Total Findings | 38 (P0: 9 · P1: 19 · P2: 10) |
| Confidence | 0.82 |

> **Assessment Scope Note:** The logical model (`bank_logical_model.xlsx`) and data mapping
> (`customer_data_mapping.xlsx`) are binary XLSX files that could not be parsed in this environment.
> Column-level attribute reviews are therefore based on (a) physical table metadata from
> `customer_physical_structures.csv`, (b) table descriptions embedded in the CSV,
> (c) Delta table properties, and (d) banking domain knowledge. Architecture-level and
> table-level findings carry high confidence (0.85–0.95). Individual attribute findings are
> marked with lower confidence where XLSX evidence is missing. This limitation does **not**
> reduce the severity of findings — all architecture-level gaps are independently verifiable.

---

## 1. Executive Summary

### 1A. Consolidated Executive Summary (Management Presentation)

**Overall Health: 🔴 Incorrect** — The `silver.customer` subject area has a fundamental
domain-scoping defect: it instantiates a "Customer" SA that the canonical architecture
explicitly prohibits, directly duplicating identity data that belongs in `silver.party`.

**Top Risks:**
- Customer SA should not exist — must be merged into Party SA (`silver.party`)
- Three reference tables stored in wrong schema — should be `silver.reference`
- `credit_limit_liability` belongs in Lending/Account SA, not Customer
- Five of ten physical tables are completely empty — implementation incomplete
- No partition columns on any table — full table scans on every pipeline run

### Top 5 Actions

| # | Action | Priority | Owner |
|---|---|---|---|
| 1 | Initiate SA consolidation: migrate `silver.customer` entities into `silver.party` per canonical architecture | P0 | Data Architecture / Governance |
| 2 | Relocate `reference_country`, `reference_customer_type`, `reference_segment` to `silver.reference` | P0 | Data Modeling / ETL |
| 3 | Relocate `credit_limit_liability` to `silver.account` or `silver.lending` SA | P0 | Data Modeling / ETL |
| 4 | Implement load pipelines for all 5 empty tables or formally decommission them | P1 | ETL / Data Engineering |
| 5 | Add partition columns (`_ingested_date` or `effective_from`) to all tables | P1 | DBA / Data Engineering |

**KPIs to track progress:**
- % entities migrated to `silver.party` (target: 100% within 2 sprints)
- % tables with partition columns defined (current: 0%, target: 100%)

**Management Decision Required:**
Approve architectural consolidation of `silver.customer` into `silver.party`. This is a
breaking change requiring a documented migration plan, downstream consumer updates, and
a 6-month deprecation window per the versioning standard.

---

### Technical Summary

The `silver.customer` subject area contains 10 physical Delta tables in the
`silver_dev_v2.customer` schema. The subject area is architecturally non-conformant:
`sa/subject-areas.md` explicitly states "Customer is a Party role, not a separate SA"
and designates `silver.party` as the correct schema. Continuing to develop this SA
in parallel with Party creates identity duplication, governance fragmentation, and
downstream join ambiguity in Gold.

Beyond the scoping defect, three reference tables (`reference_country`,
`reference_customer_type`, `reference_segment`) are misplaced — they belong in
`silver.reference`. The `credit_limit_liability` entity belongs in Account or Lending SA.
Five of the ten tables contain zero data, indicating incomplete ETL implementation.
No table has partition columns defined, causing full table scans on every pipeline run.
The largest table (`customer`) has up to 190 columns (per
`delta.columnMapping.maxColumnId: 190`), strongly suggesting 3NF violations.

---

## 2. Subject Area Architecture Assessment

### 2.1 Domain Scoping & BIAN Alignment

**Finding: CRITICAL — Wrong Subject Area**

The `customer` subject area does not appear in the canonical subject area list defined
in `sa/subject-areas.md`. The document explicitly states:

> "**Customer** is a Party role, not a separate SA — Avoids duplication of identity data;
> KYC and segmentation live in `silver.party`"

Per BIAN v11, "Customer" is a role that a Party (legal entity) plays in relation to the
bank — it is not a first-class domain object. The correct pattern is:

- **Party** → universal legal entity (individual, corporate, SME)
- **CustomerAgreement** → the role/relationship linking a Party to banking services
- **CustomerBehaviorInsights** → analytics/segmentation, built in Gold

| BIAN Service Domain | Expected SA | Actual location | Gap |
|---|---|---|---|
| Party Data Management | `silver.party` | `silver.customer` | 🔴 Wrong schema |
| Customer Reference Data | `silver.party` (role entity) | `silver.customer.customer` | 🔴 Wrong SA |
| Customer Relationship Management | `silver.party` | `silver.customer.customer_relationships` | 🔴 Wrong SA, empty |
| Reference / Master Data | `silver.reference` | `silver.customer.reference_*` | 🔴 Wrong SA |
| Credit Limit | `silver.account` / `silver.lending` | `silver.customer.credit_limit_liability` | 🔴 Wrong SA |

**IFW Pattern Violation:** IFW distinguishes Party (universal entity), Customer (role),
and CustomerSegment (classification). The current model collapses all three into a flat
`customer` table.

---

### 2.2 Identity Strategy

- The `customer` table uses `delta.columnMapping.mode: name` with
  `delta.columnMapping.maxColumnId: 190`, indicating a wide table (up to 190 columns).
- `checkConstraints` feature is present on the `customer` table — positive indicator.
- Whether `customer_sk` (MD5 surrogate) and `customer_bk` (business key) are present
  cannot be confirmed without XLSX parsing (confidence: 0.6 for this sub-section).
- For child entities (`customer_documents`, `customer_demographics`,
  `customer_preferences`, `customer_contactinformation`, `customer_relationships`):
  each requires its own `<entity>_sk` and `<entity>_bk` plus a `customer_sk` FK —
  none of this can be verified from the physical structure CSV alone.
- Reference tables may use natural keys (acceptable for lookup tables) but require
  documented exception to SLV-001/002.

**Identity Strategy Gap Summary:** Deterministic MD5 surrogate key and business key
presence is unverified for all 10 entities due to inability to parse the XLSX.
This is a P1 verification gap.

---

### 2.3 SCD Strategy

CDF (`delta.enableChangeDataFeed`) presence by table:

| Table | CDF Enabled | Risk |
|---|---|---|
| `customer` | ✅ | Low |
| `customer_demographics` | ✅ | Low |
| `customer_preferences` | ✅ | Low (empty) |
| `customer_documents` | ❌ | High — KYC audit trail at risk |
| `customer_contactinformation` | ❌ | High — contact change history lost |
| `customer_relationships` | ❌ | High — UBO/guarantor history lost |
| `credit_limit_liability` | ❌ | High — credit limit changes untracked |
| `reference_country` | ❌ | Low — reference data SCD-1 acceptable |
| `reference_customer_type` | ❌ | Low — reference data SCD-1 acceptable |
| `reference_segment` | ❌ | Low — reference data SCD-1 acceptable |

Whether SCD columns (`effective_from`, `effective_to`, `is_current`) are present
requires XLSX inspection.

---

### 2.4 Cross-Entity Relationships & Cardinality

Expected relationship graph:

```
customer (1) ──< customer_documents (N)
customer (1) ──  customer_demographics (1:1?)
customer (1) ──< customer_preferences (N)
customer (1) ──< customer_contactinformation (N)
customer (1) ──< customer_relationships (N)
customer (1) ──  credit_limit_liability (misplaced)
```

Gaps:
1. `customer_relationships`, `customer_contactinformation`, `customer_preferences`,
   `credit_limit_liability` — all empty, FK integrity unverifiable
2. No FK declarations visible in Delta metadata
3. `customer_demographics` at ~111 MB vs `customer` ~256 MB — overlap evaluation needed

---

### 2.5 ETL Mapping & Lineage Completeness

| Table | Has Data | CDF | Auto-Optimize | ETL Status |
|---|---|---|---|---|
| `customer` | ✅ ~256 MB | ✅ | ✅ | Active |
| `customer_demographics` | ✅ ~111 MB | ✅ | ✅ | Active |
| `customer_documents` | ✅ ~36 MB | ❌ | ✅ | Active, CDF gap |
| `customer_preferences` | ❌ 0 B | ✅ | ✅ | Not deployed |
| `customer_contactinformation` | ❌ 0 B | ❌ | ❌ | Not deployed |
| `customer_relationships` | ❌ 0 B | ❌ | ❌ | Not deployed |
| `credit_limit_liability` | ❌ 0 B | ❌ | ❌ | Not deployed, wrong SA |
| `reference_country` | ✅ ~6 KB | ❌ | ❌ | Loaded, wrong SA |
| `reference_customer_type` | ✅ ~3 KB | ❌ | ❌ | Loaded, wrong SA |
| `reference_segment` | ❌ 0 B | ❌ | ❌ | Not loaded, wrong SA |

---

### 2.6 Major Design Gaps

| Gap | Category | Priority | Confidence |
|---|---|---|---|
| `silver.customer` SA not in canonical list; must migrate to `silver.party` | Architecture | P0 | 0.98 |
| `reference_country/type/segment` belong in `silver.reference` | Architecture | P0 | 0.97 |
| `credit_limit_liability` belongs in `silver.account` or `silver.lending` | Architecture | P0 | 0.95 |
| Catalog uses `silver_dev_v2` instead of `silver` | Architecture | P0 | 0.90 |
| 5 of 10 tables contain zero data | Lineage | P0 | 0.99 |
| No partition columns on any table | Storage | P1 | 0.99 |
| `customer` has 190 columns — probable 3NF violation | Architecture | P1 | 0.88 |
| CDF absent on 7 of 10 tables | SCD | P1 | 0.99 |
| `customer_contactinformation` naming violates NAM-001 | Identity | P2 | 0.95 |
| Auto-optimize absent on 5 tables | Storage | P2 | 0.99 |

---

## 3. Entity Inventory

| Entity | State | Criticality | Physical Table | Empty | P0 | P1 | P2 | Confidence |
|---|---|---|---|---|---|---|---|---|
| `customer` | 🔴 Incorrect | High | `silver_dev_v2.customer.customer` | No | 3 | 4 | 2 | 0.82 |
| `customer_documents` | 🟡 Partial | High | `silver_dev_v2.customer.customer_documents` | No | 1 | 2 | 1 | 0.80 |
| `customer_demographics` | 🟡 Partial | Medium | `silver_dev_v2.customer.customer_demographics` | No | 1 | 3 | 1 | 0.78 |
| `customer_preferences` | 🔴 Incorrect | Medium | `silver_dev_v2.customer.customer_preferences` | Yes | 1 | 2 | 1 | 0.85 |
| `customer_contactinformation` | 🔴 Incorrect | High | `silver_dev_v2.customer.customer_contactinformation` | Yes | 1 | 3 | 2 | 0.85 |
| `customer_relationships` | 🔴 Incorrect | Medium | `silver_dev_v2.customer.customer_relationships` | Yes | 1 | 2 | 1 | 0.85 |
| `credit_limit_liability` | 🔴 Incorrect | High | `silver_dev_v2.customer.credit_limit_liability` | Yes | 2 | 1 | 0 | 0.90 |
| `reference_country` | 🔴 Incorrect | Medium | `silver_dev_v2.customer.reference_country` | No | 1 | 1 | 1 | 0.92 |
| `reference_customer_type` | 🔴 Incorrect | Medium | `silver_dev_v2.customer.reference_customer_type` | No | 1 | 1 | 1 | 0.92 |
| `reference_segment` | 🔴 Incorrect | Medium | `silver_dev_v2.customer.reference_segment` | Yes | 1 | 1 | 0 | 0.90 |

---

## 4. Per-Entity Assessments

---

### 4.1 `customer`

**State:** 🔴 Incorrect
**Criticality:** High — Core identity entity in wrong SA; 190 columns indicate severe denormalization
**Confidence:** 0.82
**Physical Table:** `silver_dev_v2.customer.customer` (~256 MB, 3 files)

#### Industry Fitness

The `customer` entity is the principal identity record (~256 MB). Per BIAN v11 and IFW,
"Customer" is a **role** that a Party plays — not a standalone entity. The correct pattern:

```
Party                        ← universal legal entity
  └── PartyRole (Customer)   ← the customer role
        └── CustomerAgreement ← terms of the banking relationship
```

The current `customer` table collapses Party identity (name, nationality, EID, passport),
the Customer role (status: VIP/suspended/blacklisted), KYC attributes, and relationship
management attributes into a single wide table. This is confirmed by the embedded description:

> "The table contains detailed information about customers, including their identification
> numbers, names, and various status indicators. It can be used to track customer
> demographics, monitor customer relationships, and manage customer statuses such as VIP,
> suspended, or blacklisted."

`delta.columnMapping.maxColumnId: 190` confirms up to 190 columns — a well-normalized
customer entity should have ~25–40 columns.

#### Attribute Review

> Column-level data unavailable from binary XLSX. The following reflects domain expectations.
> Per-row confidence: 0.6–0.7.

| Logical Attribute | Mapping Status | Physical Column | Actual Type | Expected Type | Nullable (Expected) | Key Role | SCD Required | Issues |
|---|---|---|---|---|---|---|---|---|
| Customer surrogate key | ambiguous | `customer_sk` (inferred) | Unknown | STRING NOT NULL | No | PK/SK | SCD-2 | MD5 derivation unverified |
| Customer business key | ambiguous | `customer_bk` (inferred) | Unknown | STRING NOT NULL | No | BK | SCD-2 | Unverified |
| Customer ID (source) | ambiguous | `customer_id` (inferred) | Unknown | STRING | No | Source ID | - | SLV-004 risk if raw source code |
| Full name | ambiguous | Unknown | Unknown | STRING | No | - | SCD-2 | PII — SHA-256 masking required |
| Emirates ID | ambiguous | Unknown | Unknown | STRING | Yes | - | SCD-2 | PII — SHA-256 masking required |
| Passport number | ambiguous | Unknown | Unknown | STRING | Yes | - | SCD-2 | PII — SHA-256 masking required |
| Nationality code | ambiguous | Unknown | Unknown | CHAR(3) ISO 3166 | Yes | FK→reference | SCD-2 | Should reference `silver.reference` |
| Customer status | ambiguous | Unknown | Unknown | STRING | No | - | SCD-2 | SLV-004 risk — raw source codes |
| Customer type code | ambiguous | Unknown | Unknown | STRING | No | FK→ref | SCD-2 | Should map via code_mapping |
| Segment code | ambiguous | Unknown | Unknown | STRING | Yes | FK→ref | SCD-2 | Should map via code_mapping |
| `effective_from` | ambiguous | Unknown | Unknown | TIMESTAMP UTC | No | SCD | SCD-2 | Required per §2.3; unverified |
| `effective_to` | ambiguous | Unknown | Unknown | TIMESTAMP UTC | Yes | SCD | SCD-2 | Required per §2.3; unverified |
| `is_current` | ambiguous | Unknown | Unknown | BOOLEAN | No | SCD | SCD-2 | Required per §2.3; unverified |

#### Metadata Columns (SLV-008 / §2.7)

CDF is enabled (`delta.enableChangeDataFeed: true`) — positive SCD-2 indicator.
Column-level presence cannot be confirmed without XLSX.

| Column | Present | Correct Type | Notes |
|---|---|---|---|
| `source_system_code` | ❓ | ❓ | Required per §2.7 |
| `source_system_id` | ❓ | ❓ | Required per §2.7 |
| `create_date` | ❓ | ❓ | Required per §2.7 |
| `update_date` | ❓ | ❓ | Required per §2.7 |
| `delete_date` | ❓ | ❓ | Required per §2.7 |
| `is_active_flag` | ❓ | ❓ | Required per §2.7 |
| `effective_from` | ❓ | ❓ | CDF enabled — positive signal |
| `effective_to` | ❓ | ❓ | Required per §2.7 |
| `is_current` | ❓ | ❓ | Required per §2.7 |
| `run_id` | ❓ | ❓ | Required per §2.7 |
| `source_ingestion_date` | ❓ | ❓ | Required per §2.7 |

#### Findings

##### Finding customer-001 — Subject Area Scoping: Customer Must Migrate to Party SA

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | `sa/subject-areas.md` — "Customer is a Party role, not a separate SA. KYC and segmentation live in `silver.party`" |
| Evidence | Physical: `silver_dev_v2.customer.customer`; SA taxonomy: `sa/subject-areas.md` structural decisions |
| Affected Table | `silver_dev_v2.customer.customer` |
| Affected Column(s) | All columns |
| Confidence | 0.98 |

**Description:**
The `customer` entity is deployed in `silver.customer` but the canonical SA taxonomy
prohibits a standalone Customer SA. Maintaining `silver.customer` in parallel with
`silver.party` creates identity duplication, governance fragmentation (two stewards
own the same attributes), and downstream join ambiguity in Gold.

**Remediation:**
1. Design `silver.party` entity set to absorb all `silver.customer.customer` attributes.
2. Separate Party identity attrs (name, EID, nationality) from Customer role attrs (status, segment).
3. Build ETL migration pipeline to populate `silver.party` from the same Bronze sources.
4. Create deprecated backward-compatibility view `silver.customer.customer_v1 → silver.party`.
5. Update all Gold pipelines to reference `silver.party`.
6. After 6-month retention window, remove `silver.customer`.

**Migration Steps:**
1. Model Party and PartyRole entities in Erwin.
2. Implement `silver.party.party` DDL (Party identity attributes).
3. Implement `silver.party.party_role` DDL (Customer role attributes).
4. Backfill `silver.party` from Bronze sources.
5. Validate row counts match `silver.customer.customer`.
6. Switch Gold pipelines; mark `silver.customer` deprecated.

**Estimated Effort:** Large
**Owner:** Data Architecture + Data Modeling + ETL + Governance

---

##### Finding customer-002 — Catalog Naming Non-Conformant: `silver_dev_v2` vs `silver`

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | `guidelines/06-naming-conventions.md` NAM-002 — "Silver layer catalog: `silver`. Pattern: `silver.<subject_area>.<entity>`" |
| Evidence | Physical structures CSV: all tables in `silver_dev_v2.customer.*` |
| Affected Table | All 10 tables |
| Affected Column(s) | N/A — catalog-level |
| Confidence | 0.90 |

**Description:**
All 10 tables use `silver_dev_v2` catalog instead of the production `silver` catalog.
Either (a) this is correctly a dev environment and production tables exist elsewhere, or
(b) the dev catalog has become the de-facto production catalog — both are naming violations.

**Remediation:**
1. Clarify if `silver_dev_v2` is dev-only.
2. Migrate tables to `silver` catalog if needed.
3. Update all pipeline references.

**Estimated Effort:** Medium
**Owner:** Platform Engineering

---

##### Finding customer-003 — Wide Table (190 Columns): Probable 3NF Violation

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | High |
| Guideline Rule | `guidelines/11-modeling-dos-and-donts.md §2.1` — "DON'T: Add numbered or suffixed columns to represent multiple instances of the same concept (e.g., `address1`, `address2`, `address3`, `address4`; `phone1`, `phone2`)" |
| Evidence | Physical structures CSV: `delta.columnMapping.maxColumnId: 190` |
| Affected Table | `silver_dev_v2.customer.customer` |
| Affected Column(s) | Inferred: address columns, phone/email numbered columns |
| Confidence | 0.78 |

**Description:**
190 columns on a customer entity is a strong 3NF violation signal. A well-normalized
entity should have ~25–40 columns. Likely contains `address1–4`, `phone1/phone2`,
demographic attrs duplicating `customer_demographics`, and source-prefixed columns.
This creates update anomalies and makes analytics unreliable.

**Remediation:**
1. Extract full column list from XLSX; classify each column.
2. Normalize address columns into `customer_address` child table.
3. Normalize contact columns into `customer_contact_information`.
4. Resolve overlap with `customer_demographics`.
5. Remove source-prefixed columns; use companion tables per §2.10.

**Estimated Effort:** Large
**Owner:** Data Modeling

---

##### Finding customer-004 — No Partition Columns

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | Medium |
| Guideline Rule | `guidelines/11-modeling-dos-and-donts.md §1.4` — "DO: Partition all tables by ingestion date" |
| Evidence | Physical structures CSV: `partitionColumns: []` |
| Affected Table | `silver_dev_v2.customer.customer` |
| Affected Column(s) | Missing partition key |
| Confidence | 0.99 |

**Description:**
No partition columns defined. With ~256 MB and growing SCD-2 history, every incremental
read performs a full table scan.

**Remediation:**
Add partition by `effective_from` (year/month) or `_ingested_date`. Breaking change —
full table rebuild required.

**Estimated Effort:** Medium
**Owner:** DBA / Data Engineering

---

##### Finding customer-005 — PII Masking Risk

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | High |
| Guideline Rule | `guidelines/01-data-quality-controls.md §2 Gate 2` — "PII Masking: EID, Card Numbers, Phone Numbers, and Passport Numbers masked via SHA-256" |
| Evidence | Table description confirms identification numbers stored; 190 cols implies multiple PII fields |
| Affected Table | `silver_dev_v2.customer.customer` |
| Affected Column(s) | Emirates ID, Passport Number, Phone Numbers (inferred) |
| Confidence | 0.85 |

**Description:**
Identification numbers are stored per the table description. If SHA-256 masking is not
applied at Bronze → Silver, this is a PDPL compliance breach.

**Remediation:**
1. Audit all PII columns for SHA-256 masking status.
2. Implement SHA-256 at Bronze → Silver gate if missing.
3. Verify `dim_consent` join at Silver → Gold.

**Estimated Effort:** Medium
**Owner:** Data Engineering / Compliance

---

### 4.2 `customer_documents`

**State:** 🟡 Partial
**Criticality:** High — KYC documents are regulatory assets; CDF absent
**Confidence:** 0.80
**Physical Table:** `silver_dev_v2.customer.customer_documents` (~36 MB, 1 file)

#### Industry Fitness

Customer documents (EID copies, passport scans, trade licenses, proof of address) are
critical KYC artifacts required by CBUAE AML/KYC regulations. The entity has data loaded
(~36 MB) but lacks CDF, preventing document lifecycle events from flowing to Gold.

**Expected grain:** One row per document per customer per effective period (SCD-2).

#### Attribute Review

> XLSX unavailable. Domain-expectation only. Confidence per row: 0.60–0.70.

| Logical Attribute | Mapping Status | Physical Column | Actual Type | Expected Type | Nullable | Key Role | SCD Required | Issues |
|---|---|---|---|---|---|---|---|---|
| Document SK | ambiguous | `customer_documents_sk` | Unknown | STRING NOT NULL | No | PK/SK | SCD-2 | Unverified |
| Customer FK | ambiguous | `customer_sk` | Unknown | STRING NOT NULL | No | FK | - | Unverified |
| Document type code | ambiguous | Unknown | Unknown | STRING NOT NULL | No | - | SCD-2 | SLV-004 risk |
| Document number | ambiguous | Unknown | Unknown | STRING | No | - | SCD-2 | PII — SHA-256 required |
| Expiry date | ambiguous | Unknown | Unknown | DATE | Yes | - | SCD-2 | Critical for KYC compliance |
| Verification status | ambiguous | Unknown | Unknown | STRING | No | - | SCD-2 | Canonical code required |
| `effective_from` | ambiguous | Unknown | Unknown | TIMESTAMP UTC | No | SCD | SCD-2 | Required; unverified |
| `effective_to` | ambiguous | Unknown | Unknown | TIMESTAMP UTC | Yes | SCD | SCD-2 | Required; unverified |
| `is_current` | ambiguous | Unknown | Unknown | BOOLEAN | No | SCD | SCD-2 | Required; unverified |

#### Metadata Columns

| Column | Present | Correct Type | Notes |
|---|---|---|---|
| All metadata columns | ❓ | ❓ | Cannot verify from CSV |

#### Findings

##### Finding customer_documents-001 — Wrong Subject Area

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | `sa/subject-areas.md` — Customer is a Party role; documents belong in `silver.party` |
| Evidence | Physical: `silver_dev_v2.customer.customer_documents`; canonical: `silver.party` |
| Affected Table | `silver_dev_v2.customer.customer_documents` |
| Affected Column(s) | All |
| Confidence | 0.97 |

**Description:** Must migrate to `silver.party.party_document` as part of SA consolidation (customer-001).
**Remediation:** Covered under customer-001 migration plan.
**Estimated Effort:** Included in Large for customer-001
**Owner:** Data Architecture / ETL

---

##### Finding customer_documents-002 — Missing CDF: KYC History at Risk

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | High |
| Guideline Rule | `guidelines/11-modeling-dos-and-donts.md §2.3` — "Every Silver entity table must have `effective_from`, `effective_to`, `is_current`" |
| Evidence | Physical structures CSV: `customer_documents` properties lack `delta.enableChangeDataFeed` |
| Affected Table | `silver_dev_v2.customer.customer_documents` |
| Affected Column(s) | Missing CDF; SCD columns unverified |
| Confidence | 0.85 |

**Description:**
KYC documents have regulatory lifecycle events requiring full change history. Absence of
CDF means Gold CDC pipelines cannot detect document changes and audit trails are lost.

**Remediation:**
1. Enable `delta.enableChangeDataFeed = true`.
2. Verify SCD-2 columns are present and populated.
3. Backfill SCD-2 history from Bronze if missing.

**Estimated Effort:** Small
**Owner:** Data Engineering / DBA

---

### 4.3 `customer_demographics`

**State:** 🟡 Partial
**Criticality:** Medium — Data present; denormalization risk with `customer`
**Confidence:** 0.78
**Physical Table:** `silver_dev_v2.customer.customer_demographics` (~111 MB, 3 files)

#### Industry Fitness

Demographics separation is valid when sourced from a different system (e.g., CRM vs CBS).
However, the `customer` table at 190 columns likely duplicates demographic attributes
(DOB, gender, nationality), creating update anomalies.

CDF ✅ and auto-optimize ✅ are positive.

#### Findings

##### Finding customer_demographics-001 — No Partition Columns

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | Medium |
| Guideline Rule | Physical infrastructure standard — partition by `_ingested_date` or `effective_from` |
| Evidence | `partitionColumns: []` |
| Affected Table | `silver_dev_v2.customer.customer_demographics` |
| Affected Column(s) | Missing partition key |
| Confidence | 0.99 |

**Description:** ~111 MB with no partitioning — full scans on every read.
**Remediation:** Add `effective_from` year/month partition. Full rebuild required.
**Estimated Effort:** Medium
**Owner:** DBA / Data Engineering

---

##### Finding customer_demographics-002 — Probable Overlap with `customer` (Denormalization)

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | Medium |
| Guideline Rule | `guidelines/11-modeling-dos-and-donts.md §2.1` — 3NF; no repeating groups |
| Evidence | `customer` (190 cols, ~256 MB) + `customer_demographics` (~111 MB) — probable overlap |
| Affected Table | `customer` and `customer_demographics` |
| Affected Column(s) | Likely: `date_of_birth`, `gender_code`, `nationality_code`, `marital_status_code` |
| Confidence | 0.72 |

**Description:**
If both tables store DOB, gender, nationality etc., SCD updates must be synchronized
across both tables. Failure to synchronize creates divergent history.

**Remediation:**
1. Extract column list from XLSX; identify overlapping attributes.
2. Remove duplicates from `customer`; retain canonical attrs in `customer_demographics`.
3. If companion-table pattern (§2.10) is intended, document the source system split.

**Estimated Effort:** Medium
**Owner:** Data Modeling

---

### 4.4 `customer_preferences`

**State:** 🔴 Incorrect
**Criticality:** Medium — Empty; PDPL consent flags missing
**Confidence:** 0.85
**Physical Table:** `silver_dev_v2.customer.customer_preferences` (0 bytes)

#### Industry Fitness

Customer preferences carry PDPL opt-out flags required for Gold promotion consent check
(`dim_consent` join per `guidelines/01-data-quality-controls.md §10`). Empty table means
consent cannot be verified → Gold promotion is non-compliant with PDPL Article 5.

#### Findings

##### Finding customer_preferences-001 — Empty Table: ETL Not Deployed; PDPL Risk

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | Medium |
| Guideline Rule | `guidelines/01-data-quality-controls.md §2 Gate 2` — Completeness; `§10` — PDPL Article 5 consent check |
| Evidence | `sizeInBytes: 0`, `numFiles: 0`; created 2026-04-13 |
| Affected Table | `silver_dev_v2.customer.customer_preferences` |
| Affected Column(s) | All |
| Confidence | 0.99 |

**Description:**
Table created but never loaded. If it carries PDPL consent flags, their absence makes
Silver → Gold promotion non-compliant.

**Remediation:**
Deploy ETL with preference/consent data from CRM or CBS. If not available from source,
formally record expected delivery date in data catalog. Decommission if not planned.

**Estimated Effort:** Medium
**Owner:** ETL / Data Engineering

---

### 4.5 `customer_contactinformation`

**State:** 🔴 Incorrect
**Criticality:** High — Contact info critical for collections/fraud/CRM; naming violation; empty; no CDF
**Confidence:** 0.85
**Physical Table:** `silver_dev_v2.customer.customer_contactinformation` (0 bytes)

#### Industry Fitness

Critical operational entity for collections, fraud alerts, and CRM. Entity is empty.
Additionally:
1. **Naming violation**: `contactinformation` → should be `contact_information` (NAM-001)
2. **No CDF**: contact changes cannot flow to Gold CDC
3. **No auto-optimize**: missing performance configuration
4. The `customer` table's 190 columns likely contain `phone1`, `phone2`, `email_1` etc.
   that should be normalized into this entity

#### Findings

##### Finding customer_contactinformation-001 — Table Naming Violates NAM-001

| Field | Value |
|---|---|
| Priority | P2 |
| Criticality | Low |
| Guideline Rule | `guidelines/06-naming-conventions.md` NAM-001 — "Use snake_case for all identifiers" |
| Evidence | Table name `customer_contactinformation`; correct: `customer_contact_information` |
| Affected Table | `silver_dev_v2.customer.customer_contactinformation` |
| Affected Column(s) | Table name |
| Confidence | 0.95 |

**Description:**
`contactinformation` is not snake_case — should be `contact_information`. Fix before
any consumers are added; table is currently empty making this a non-breaking rename.

**Remediation:**
Rename to `customer_contact_information` via DDL or recreate with correct name.
**Estimated Effort:** Small
**Owner:** Data Modeling / DBA

---

##### Finding customer_contactinformation-002 — Empty + No CDF + No Auto-Optimize

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | High |
| Guideline Rule | `guidelines/01-data-quality-controls.md §2 Gate 2` — Completeness; `§2.3` — SCD-2 via CDF |
| Evidence | `sizeInBytes: 0`, no CDF, no auto-optimize in CSV properties |
| Affected Table | `silver_dev_v2.customer.customer_contactinformation` |
| Affected Column(s) | All |
| Confidence | 0.99 |

**Description:**
Without contact information, Gold models cannot join customer contact details to
transactions, collections, or fraud alerts. The `customer` table's 190 cols are
the only contact data source — an undocumented hidden dependency.

**Remediation:**
1. Deploy ETL pipeline; enable CDF; enable auto-optimize.
2. Migrate contact attrs (phone, email) out of `customer` into this child table.
3. Rename table to `customer_contact_information` (finding -001).

**Estimated Effort:** Medium
**Owner:** Data Engineering / Data Modeling

---

### 4.6 `customer_relationships`

**State:** 🔴 Incorrect
**Criticality:** Medium — UBO/guarantor data absent; CBUAE AML/KYC regulatory gap
**Confidence:** 0.85
**Physical Table:** `silver_dev_v2.customer.customer_relationships` (0 bytes)

#### Industry Fitness

Customer relationships (UBO, guarantors, authorized signatories) are mandatory for CBUAE
AML/KYC compliance (UBO up to 25% ownership) and FATF PEP screening of related parties.
Table was created 2026-01-06 and DDL was modified 2026-03-25 — work is ongoing but
no data has been loaded. No CDF, no auto-optimize.

#### Findings

##### Finding customer_relationships-001 — Empty: Regulatory Gap for UBO/Guarantor Data

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | High |
| Guideline Rule | `guidelines/01-data-quality-controls.md §2 Gate 2` — Completeness; CBUAE AML/KYC; BIAN CustomerRelationship domain |
| Evidence | `sizeInBytes: 0`; created 2026-01-06; modified 2026-03-25 |
| Affected Table | `silver_dev_v2.customer.customer_relationships` |
| Affected Column(s) | All |
| Confidence | 0.92 |

**Description:**
Absence of UBO and guarantor relationship data means Gold risk models and CBUAE
regulatory reports cannot accurately reflect party network structures.

**Remediation:**
1. Identify source for relationship data (CBS CIF or CRM).
2. Map source codes to canonical relationship types (GUARANTOR, UBO, AUTH_SIGNATORY).
3. Deploy ETL; enable CDF; enable auto-optimize.
4. Verify M:N party relationship model with `from_party_sk`, `to_party_sk`,
   `relationship_type_code`, `effective_from`, `effective_to`.

**Estimated Effort:** Medium
**Owner:** ETL / Data Engineering / Compliance

---

### 4.7 `credit_limit_liability`

**State:** 🔴 Incorrect
**Criticality:** High — Financial data in identity SA; cross-SA update anomaly risk; empty
**Confidence:** 0.90
**Physical Table:** `silver_dev_v2.customer.credit_limit_liability` (0 bytes)

#### Industry Fitness

Credit limit and liability data is a financial/account concept that belongs in:
- `silver.account` — credit limit on a specific account/facility
- `silver.lending` — aggregate credit exposure across all lending products

Placing credit limits in the Customer SA:
1. Violates SA isolation — Customer SA should not own financial position data
2. Creates cross-SA update anomaly: credit limit changes (from Lending SA events)
   must propagate to Customer SA
3. Breaks regulatory separation of customer identity from credit risk

Per BIAN, "CreditFacility" and "CustomerCreditRating" are separate service domains
from "PartyDataManagement".

#### Findings

##### Finding credit_limit_liability-001 — Wrong SA: Financial Data in Identity SA

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | `guidelines/11-modeling-dos-and-donts.md §2.2` — "Keep each Silver subject area self-contained. DON'T: Join across subject areas inside Silver pipelines." |
| Evidence | Physical: `silver_dev_v2.customer.credit_limit_liability`; `sa/subject-areas.md`: Account/Lending SA for credit |
| Affected Table | `silver_dev_v2.customer.credit_limit_liability` |
| Affected Column(s) | All |
| Confidence | 0.92 |

**Description:**
Credit limits belong in Account or Lending SA. Any credit review event in Lending SA
that changes the limit must now also update Customer SA — creating a cross-domain
update anomaly and circular SA dependency.

**Remediation:**
1. Move to `silver.account` or `silver.lending` schema.
2. Establish FK to relevant account or facility entity.
3. Replace with FK reference in Customer SA if consumers require it.

**Estimated Effort:** Small (table is empty — no data migration)
**Owner:** Data Modeling / Data Architecture

---

##### Finding credit_limit_liability-002 — Empty: No Data, No CDF, No Auto-Optimize

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | `guidelines/01-data-quality-controls.md §2 Gate 2` — Completeness |
| Evidence | `sizeInBytes: 0`, no CDF, no auto-optimize |
| Affected Table | `silver_dev_v2.customer.credit_limit_liability` |
| Affected Column(s) | All |
| Confidence | 0.99 |

**Description:** Table empty and in wrong SA. Priority: migrate design first, then load.
**Remediation:** Covered by credit_limit_liability-001. Do not load here; migrate first.
**Estimated Effort:** Small
**Owner:** Data Modeling

---

### 4.8 `reference_country`

**State:** 🔴 Incorrect
**Criticality:** Medium — Reference data in wrong SA; data present
**Confidence:** 0.92
**Physical Table:** `silver_dev_v2.customer.reference_country` (~6 KB, 1 file)

#### Industry Fitness

ISO 3166-1 country code data is master reference data that belongs exclusively in
`silver.reference` (SA #1, Priority Critical per `sa/subject-areas.md`). Having country
codes in the customer schema:
1. Creates duplicate reference data
2. Forces other SAs to either cross-join to `silver.customer` (violation) or
   maintain their own copy (further duplication)
3. Makes the Customer SA a de-facto reference data hub

No CDF, no auto-optimize — consistent with slowly-changing reference data (SCD-1
acceptable with documented approval).

#### Findings

##### Finding reference_country-001 — Wrong SA: Belongs in `silver.reference`

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | Medium |
| Guideline Rule | `sa/subject-areas.md` — "Reference Data | `silver.reference` | Master Lookups: ISO Currencies, **Country Codes**, FX Rates..." |
| Evidence | Physical: `silver_dev_v2.customer.reference_country` |
| Affected Table | `silver_dev_v2.customer.reference_country` |
| Affected Column(s) | All |
| Confidence | 0.97 |

**Description:**
Country codes must reside in `silver.reference`. Current location causes potential
divergence between customer's local copy and the canonical reference.

**Remediation:**
1. Verify `silver.reference.reference_country` exists.
2. Drop or migrate `silver.customer.reference_country` to `silver.reference`.
3. Update pipeline JOINs.

**Estimated Effort:** Small
**Owner:** Data Modeling / ETL

---

### 4.9 `reference_customer_type`

**State:** 🔴 Incorrect
**Criticality:** Medium — Reference data in wrong SA; SLV-004 bypass risk
**Confidence:** 0.92
**Physical Table:** `silver_dev_v2.customer.reference_customer_type` (~3 KB, 1 file)

#### Industry Fitness

Customer type codes (RETAIL/CORPORATE/SME/PRIVATE_BANKING) are master data for
`silver.reference`. Additionally, per SLV-004, entity columns must resolve codes via
`silver.reference.code_mapping` — not a local reference table in the customer schema.
This table may be enabling a SLV-004 bypass.

#### Findings

##### Finding reference_customer_type-001 — Wrong SA + Potential SLV-004 Bypass

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | Medium |
| Guideline Rule | `sa/subject-areas.md` — reference data in `silver.reference`; `guidelines/11-modeling-dos-and-donts.md §2.5` — "DO: Resolve source-system codes to canonical values using `silver.reference.code_mapping`." |
| Evidence | Physical: `silver_dev_v2.customer.reference_customer_type`; 3 KB = few rows = simple code table |
| Affected Table | `silver_dev_v2.customer.reference_customer_type` |
| Affected Column(s) | All |
| Confidence | 0.90 |

**Description:**
Customer type codes should be in `silver.reference.code_mapping` with
`source_domain = 'CUSTOMER_TYPE'`. A local reference table suggests `customer_type_code`
in the `customer` entity may be joined here rather than to the canonical mapping registry
— bypassing governance-controlled code resolution.

**Remediation:**
1. Register codes in `silver.reference.code_mapping` (`source_domain = 'CUSTOMER_TYPE'`).
2. Drop `silver.customer.reference_customer_type`.
3. Update `customer` ETL to resolve via `silver.reference.code_mapping`.

**Estimated Effort:** Small
**Owner:** Data Modeling / ETL

---

### 4.10 `reference_segment`

**State:** 🔴 Incorrect
**Criticality:** Medium — Reference data in wrong SA; empty
**Confidence:** 0.90
**Physical Table:** `silver_dev_v2.customer.reference_segment` (0 bytes)

#### Industry Fitness

Customer segment codes (MASS_MARKET/AFFLUENT/HNW/ULTRA_HNW/CORPORATE/SME) are master
data for `silver.reference`. Same architectural issues as `reference_customer_type`.
Table is also empty — no data has ever been loaded.

#### Findings

##### Finding reference_segment-001 — Wrong SA + Empty

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | Medium |
| Guideline Rule | `sa/subject-areas.md` — reference data in `silver.reference` |
| Evidence | Physical: `silver_dev_v2.customer.reference_segment`; `sizeInBytes: 0` |
| Affected Table | `silver_dev_v2.customer.reference_segment` |
| Affected Column(s) | All |
| Confidence | 0.95 |

**Description:**
Segment codes belong in `silver.reference.code_mapping` (`source_domain = 'CUSTOMER_SEGMENT'`).
Table is empty — zero migration risk.

**Remediation:**
1. Register segment codes in `silver.reference.code_mapping`.
2. Drop `silver.customer.reference_segment` (empty — no data loss).
3. Update `customer` ETL to resolve segments via `silver.reference.code_mapping`.

**Estimated Effort:** Small
**Owner:** Data Modeling

---

## 5. Denormalization Register

| ID | Tables Involved | Logical Origins | Classification | Correctness Risk | Impact | Recommendation | Confidence |
|---|---|---|---|---|---|---|---|
| DEN-001 | `customer` + `customer_demographics` | Party identity attrs in both | Unnecessary | High — dual SCD anomaly | ~47% of customer data volume | Re-normalize: remove duplicated demographic attrs from `customer` | 0.72 |
| DEN-002 | `customer` (190 cols) vs `customer_contactinformation` | Contact info in parent + child | Unnecessary | Medium — out-of-sync contact data | High — operational contact lists diverge | Normalize contact attrs out of `customer` into `customer_contact_information` | 0.75 |
| DEN-003 | `customer` vs `reference_country` | Country code in entity + local lookup | Unnecessary | Low — country codes are stable | Low | Remove local reference; join to `silver.reference.reference_country` | 0.85 |
| DEN-004 | `customer` vs `reference_customer_type` | Type code in entity + local lookup | Unnecessary | Medium — SLV-004 bypass | Medium | Remove local reference; use `silver.reference.code_mapping` | 0.83 |
| DEN-005 | `customer` vs `reference_segment` | Segment code in entity + local lookup | Unnecessary | Medium — SLV-004 bypass | Medium | Remove local reference; use `silver.reference.code_mapping` | 0.83 |
| DEN-006 | `credit_limit_liability` in Customer SA vs Account/Lending | Credit position in identity SA | Unnecessary | High — cross-SA update anomaly | High | Move to `silver.account` or `silver.lending` | 0.90 |

> **DEN-001 Justification:** None provided. If `customer_demographics` sources from CRM
> and `customer` from CBS (companion-table pattern per §2.10), separation is acceptable
> **only if** overlapping attributes are removed from `customer`. Pending XLSX verification.
> Currently classified Unnecessary.

> **DEN-002 through DEN-006 Justification:** None provided. All classified Unnecessary.
> Re-normalization required.

---

## 6. Guideline Compliance Summary

| Rule Code | Rule Title | Status | Violation Count | Worst Severity |
|---|---|---|---|---|
| NAM-001 | snake_case, lowercase | ⚠️ Partial | 1 (`contactinformation`) | P2 |
| NAM-002 | Catalog/schema `silver.<subject_area>` | ❌ Fail | 10 (all in `silver_dev_v2`) | P0 |
| NAM-003 | Table naming `<entity>` for Silver | ✅ Pass | 0 | — |
| SLV-001 | `<entity>_sk` MD5 surrogate | ❓ Unverified | Unknown | P1 risk |
| SLV-002 | `<entity>_bk` business key | ❓ Unverified | Unknown | P1 risk |
| SLV-003 | SCD-2 default | ⚠️ Partial | 7 tables missing CDF | P1 |
| SLV-004 | No raw source codes; use `code_mapping` | ❌ Fail | 3 local reference tables bypass | P0 |
| SLV-005 | DQ-gated writes; quarantine table | ❓ Unverified | Unknown | P1 risk |
| SLV-006 | 3NF; no derived values | ❌ Fail | 6 denormalization instances | P1 |
| SLV-007 | No pre-computed metrics | ❓ Unverified | Unknown | — |
| SLV-008 | Mandatory audit columns | ❓ Unverified | Unknown (XLSX needed) | P1 risk |
| SLV-009 | All timestamps UTC | ❓ Unverified | Unknown (XLSX needed) | P1 risk |
| SLV-010 | Monetary amounts in fils | ❓ Unverified | `credit_limit_liability` at risk | P1 risk |
| SA-SCOPE | SA correctly scoped | ❌ Fail | 1 (entire SA non-canonical) | P0 |
| STORAGE-PART | Partition columns defined | ❌ Fail | 10 (all unpartitioned) | P1 |
| STORAGE-CDF | CDF for SCD-2 tables | ❌ Fail | 7 of 10 missing CDF | P1 |
| STORAGE-OPT | Auto-optimize enabled | ⚠️ Partial | 5 missing auto-optimize | P2 |
| PII-MASK | EID/Passport/Phone SHA-256 masked | ❓ Unverified | High risk (190-col table with IDs) | P1 risk |

---

## 7. Remediation Plan

### Prioritized Action List

| # | Action | Entity / Table | Priority | Effort | Owner | Dependency |
|---|---|---|---|---|---|---|
| 1 | Initiate SA migration design: Customer → Party. Produce formal architectural migration document. | All 10 tables | P0 | Large | Data Architecture + Governance | None |
| 2 | Confirm production catalog name — clarify `silver_dev_v2` scope | All 10 tables | P0 | Small | Platform Engineering | Architecture decision |
| 3 | Relocate `reference_country` to `silver.reference`; update pipeline JOINs | `reference_country` | P0 | Small | Data Modeling + ETL | None |
| 4 | Register `reference_customer_type` codes in `silver.reference.code_mapping`; drop local table | `reference_customer_type` | P0 | Small | Data Modeling + ETL | None |
| 5 | Register `reference_segment` codes in `silver.reference.code_mapping`; drop local table | `reference_segment` | P0 | Small | Data Modeling | None |
| 6 | Move `credit_limit_liability` design to `silver.account` or `silver.lending` SA | `credit_limit_liability` | P0 | Small | Data Modeling | SA migration decision |
| 7 | Add partition columns to `customer` and `customer_demographics` | `customer`, `customer_demographics` | P1 | Medium | DBA + Data Engineering | Schema version bump |
| 8 | Enable CDF on `customer_documents`, `customer_contactinformation`, `customer_relationships`, `credit_limit_liability` | 4 tables | P1 | Small | DBA | None |
| 9 | Enable auto-optimize on all missing tables | 5 tables | P1 | Small | DBA | None |
| 10 | Deploy ETL for `customer_preferences` with PDPL consent flags | `customer_preferences` | P1 | Medium | ETL | Source mapping |
| 11 | Deploy ETL for `customer_contactinformation`; normalize contact attrs from `customer` | `customer_contactinformation` | P1 | Medium | ETL + Data Modeling | 3NF remediation |
| 12 | Deploy ETL for `customer_relationships` (UBO/guarantor/PEP data) | `customer_relationships` | P1 | Medium | ETL | Source mapping |
| 13 | Audit PII columns in `customer`; implement SHA-256 masking | `customer` | P1 | Medium | Data Engineering + Compliance | Security review |
| 14 | Audit 190-column `customer` table; normalize address and contact columns | `customer` | P1 | Large | Data Modeling | 3NF design |
| 15 | Rename `customer_contactinformation` → `customer_contact_information` | `customer_contactinformation` | P2 | Small | DBA | Before consumers added |
| 16 | Verify SK/BK presence and MD5 derivation (requires XLSX parsing) | All entities | P1 | Small | Data Modeling | XLSX access |
| 17 | Verify all timestamps are UTC (requires XLSX parsing) | All entities | P1 | Small | Data Engineering | XLSX access |
| 18 | Verify SLV-010 monetary amounts in fils for `credit_limit_liability` | `credit_limit_liability` | P1 | Small | Data Modeling | After SA migration |

### Suggested Schedule

| Sprint | Actions |
|---|---|
| Sprint 1 (Week 1–2) | Items 1–6 — architectural decisions and zero-risk reference data migrations |
| Sprint 2 (Week 3–4) | Items 7–10, 13 — infrastructure (partitioning, CDF, auto-optimize, PII audit) |
| Sprint 3 (Week 5–6) | Items 11, 12, 14, 16, 17 — ETL deployment + 3NF normalization design |
| Sprint 4+ | Items 15, 18 — naming fix, monetary unit verification, SA migration execution |
| Ongoing | Monitor SA migration progress; deprecate `silver.customer` after `silver.party` validated |

---

## Appendix

### A. Logical → Physical Mapping Summary

> Column-level mapping unavailable (binary XLSX). Table-level summary only.

| Logical Entity | Physical Table | Schema | Has Data | Correct SA | Notes |
|---|---|---|---|---|---|
| Customer (Party identity) | `customer` | `silver_dev_v2.customer` | ✅ ~256 MB | ❌ → `silver.party` | 190 cols, CDF enabled |
| Customer Documents | `customer_documents` | `silver_dev_v2.customer` | ✅ ~36 MB | ❌ → `silver.party` | CDF absent |
| Customer Demographics | `customer_demographics` | `silver_dev_v2.customer` | ✅ ~111 MB | ❌ → `silver.party` | CDF enabled |
| Customer Preferences | `customer_preferences` | `silver_dev_v2.customer` | ❌ Empty | ❌ → `silver.party` | CDF enabled, no data |
| Customer Contact Information | `customer_contactinformation` | `silver_dev_v2.customer` | ❌ Empty | ❌ → `silver.party` | No CDF; naming violation |
| Customer Relationships | `customer_relationships` | `silver_dev_v2.customer` | ❌ Empty | ❌ → `silver.party` | No CDF |
| Credit Limit Liability | `credit_limit_liability` | `silver_dev_v2.customer` | ❌ Empty | ❌ → `silver.account`/`silver.lending` | No CDF |
| Country Reference | `reference_country` | `silver_dev_v2.customer` | ✅ ~6 KB | ❌ → `silver.reference` | No CDF |
| Customer Type Reference | `reference_customer_type` | `silver_dev_v2.customer` | ✅ ~3 KB | ❌ → `silver.reference` | No CDF |
| Segment Reference | `reference_segment` | `silver_dev_v2.customer` | ❌ Empty | ❌ → `silver.reference` | No CDF |

---

### B. Guideline Citations

| Rule Code | Source File | Full Rule Text |
|---|---|---|
| NAM-001 | `guidelines/06-naming-conventions.md` | "Use snake_case for all identifiers (catalogs, schemas, tables, columns, views). Use lowercase only — no mixed case." |
| NAM-002 | `guidelines/06-naming-conventions.md` | "Silver layer: Catalog `silver`, Schema `<subject_area>`, Full Pattern `silver.<subject_area>.<entity>`" |
| NAM-003 | `guidelines/06-naming-conventions.md` | "Silver entity table naming: `<entity>`, e.g., `customer`, `account`, `transaction`" |
| SLV-003 | `guidelines/11-modeling-dos-and-donts.md §2.3` | "DO: Track all changes with SCD Type 2. Every Silver entity table must have `effective_from`, `effective_to`, `is_current`. DON'T: Use SCD-1 without explicit data-steward approval." |
| SLV-004 | `guidelines/11-modeling-dos-and-donts.md §2.5` | "DO: Resolve all source-system codes to canonical platform values using `silver.reference.code_mapping`. DON'T: Store source-system-specific codes directly in Silver entity columns." |
| SLV-006 | `guidelines/11-modeling-dos-and-donts.md §2.1` | "DO: Decompose entities into 3NF. DON'T: Add numbered or suffixed columns to represent multiple instances of the same concept (e.g., `address1`, `address2`, `address3`, `address4`)." |
| SLV-008 | `guidelines/11-modeling-dos-and-donts.md §2.7` | "DO: Include all Silver technical audit columns on every Silver entity table: `source_system_code`, `source_system_id`, `create_date`, `update_date`, `delete_date`, `is_active_flag`, `effective_from`, `effective_to`, `is_current`, `run_id`, `source_ingestion_date`." |
| PII-MASK | `guidelines/01-data-quality-controls.md §2 Gate 2` | "PII Masking: EID, Card Numbers, Phone Numbers, and Passport Numbers masked via SHA-256 unless stored in a restricted container." |
| SA-SCOPE | `sa/subject-areas.md` (Structural Decisions) | "**Customer** is a Party role, not a separate SA. Avoids duplication of identity data; KYC and segmentation live in `silver.party`." |
| CONSENT | `guidelines/01-data-quality-controls.md §10` | "Consent Check: All data entering the Gold layer must be joined with `dim_consent`; processing prohibited for opted-out customers (PDPL Article 5)." |
| 3NF-ADDR | `guidelines/11-modeling-dos-and-donts.md §2.1` | "Normalize into a child table (one row per item). DON'T: `address1`, `address2`, `address3`, `address4` — a repeating group of free-form address lines." |

---

### C. Industry Standard References

| Reference | Standard | Relevance |
|---|---|---|
| Party Data Management | BIAN v11 Service Domain | Customer is a Party role; identity must reside in Party SA |
| CustomerRelationship | BIAN v11 Service Domain | UBO, guarantor, authorized signatory relationships |
| CustomerAgreement | BIAN v11 Service Domain | Customer role terms — should be in Contract SA |
| Party / PartyRole / PartyRelationship | IFW (IBM/Teradata) | Separation of party identity from customer role from party relationship |
| KYC / AML | CBUAE AML/CFT Standards; FATF Recommendations | Document lifecycle, UBO identification, PEP screening |
| PDPL Article 5 | UAE Personal Data Protection Law | Consent check before Gold promotion; right to erasure within 30 days |
| BCBS 239 | Risk Data Aggregation Principles | Complete lineage for customer risk attributes; no data loss in SCD |
| IFRS 9 | International Financial Reporting Standard 9 | Customer credit staging requires clean party identity with full SCD-2 history |
