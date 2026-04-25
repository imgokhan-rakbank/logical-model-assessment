# Customer Subject Area — Silver Layer Gap Assessment Report

**Assessment Date**: 2025-07-14  
**Assessed By**: Logical Model Assessor (automated)  
**Subject Area**: customer  
**Physical Catalog/Schema**: `silver_dev_v2.customer` *(non-canonical — see §2.1)*  
**Canonical Schema per SA Taxonomy**: `silver.party` *(Customer is a Party role — see §2.1)*  
**Priority per SA Taxonomy**: ⚠️ **NOT LISTED** (closest canonical: Party = Priority 1 Critical)  
**Guideline Versions**: DQ Controls v1.0 Draft, Naming Conventions v1.0 Draft, Modeling DOs & DON'Ts v1.0 Draft  

---

## 1. Executive Summary

### 1.1 Management Slide

| Metric | Value |
|---|---|
| Total Logical Entities Assessed | 31 (11 core + 20 reference) |
| Physical Tables Found | 10 |
| Logical Entities Without Physical Tables | 23 |
| Orphan Physical Tables (no logical entity) | 1 (`credit_limit_liability`) |
| Total Logical Model Attributes | 584 |
| Total Mapping Rows | 375 |
| Mapped Attributes (Y) | 152 (41%) |
| Unmapped Attributes (N) | 223 (59%) |
| P0 Findings | **17** |
| P1 Findings | **14** |
| P2 Findings | **9** |
| Total Findings | **40** |
| Overall Compliance Score | **~22%** *(critical deficiencies in identity, metadata, scoping, and mapping)* |

### 1.2 Top 5 Priority Actions

1. **[P0 — Architecture]** The `customer` subject area **does not exist** in the canonical subject area taxonomy (`sa/subject-areas.md`). The taxonomy explicitly states: *"Customer is a Party role, not a separate SA — Avoids duplication of identity data; KYC and segmentation live in `silver.party`."* The entire SA must be reviewed for consolidation into `silver.party`.
2. **[P0 — Identity]** Zero entities carry a `<entity>_sk` (MD5 deterministic surrogate key) or `<entity>_bk` (business key) in the logical model. All 31 logical entities violate SLV-001 and SLV-002. The current PK candidate "Customer number (CIF)" is a descriptive label, not a `customer_bk`-named column.
3. **[P0 — Data Types]** 554 of 584 (95%) logical attributes are typed as `STRING`; 30 use `CHAR(18)`. No date attributes use `DATE`/`TIMESTAMP`, no monetary attributes use `DECIMAL(18,4)` or `BIGINT` (fils), and no boolean flags use `BOOLEAN`. This violates the standardisation requirements of DQ Controls §4 and Silver modeling rules.
4. **[P0 — Metadata]** Mandatory Silver technical audit columns are misnamed, missing, or incorrectly typed across all 11 core entities. `run_id` and `source_ingestion_date` are absent from the entire logical model. `is_current` is replaced with `IsLogicallyDeleted` (wrong semantics, wrong naming). SCD-2 columns `effective_from`/`effective_to` appear as `Business Start Date`/`Business End Date` typed `CHAR(18)` — a type error.
5. **[P0 — Mapping]** Overall mapping completeness is 41%. Five logical entities have zero mapping coverage (`Customer_Individual`, `Customer_Business`, and effectively all 20 reference entities have minimal mapping). The mapping document (`customer_data_mapping.xlsx`) contains multiple typos (`Enitity_Name`, `Customer_Relatioships`, `Souce_System.Column_Data_Type`) indicating low maturity.

---

## 2. Subject Area Architecture Assessment

### 2.1 Domain Scoping & BIAN Alignment

**Finding SA-001 | P0 | High | Confidence: 1.00**  
**Subject area `customer` is not canonical — violates SA taxonomy structural decision**

The `sa/subject-areas.md` Structural Decisions table explicitly records:

> *"**Customer** is a Party role, not a separate SA — Avoids duplication of identity data; KYC and segmentation live in `silver.party`."*

No subject area named `customer` appears in the 19-row subject area list. The physical catalog `silver_dev_v2.customer` is running in production against a design principle that forbids this separation. This creates two problems:
- **Identity duplication risk**: customer identity data duplicated across `silver.party` and `silver.customer`.
- **Consumer confusion**: downstream Gold/Semantic consumers must choose between two competing sources of truth.

**BIAN v11 alignment**: The BIAN service domain covering this scope is *Party Data Management* (domain: Party). The entities `Customer_Individual` and `Customer_Business` map to BIAN *Party* sub-types. The sub-entities `Customer_ContactInformation`, `Customer_Documents`, `Customer_Demographics`, and `Customer_Relationships` all map to BIAN *Party Reference Data* components. No distinct BIAN service domain called "Customer" exists at this level of granularity.

**IFW alignment**: IFW distinguishes *Party* (universal legal entity) from *Customer* (a role a Party plays). The current implementation conflates both, mixing party identity attributes (Full Name, DoB, Nationality) with customer-role attributes (Customer Segment Code, VIP Indicator, Primary RM Code) in the same `customer` entity — a 3NF violation.

**Remediation**: Align with `silver.party` schema. Evaluate whether `customer` SA is a transitional step toward `silver.party` or an intentional divergence requiring steward sign-off and SA taxonomy amendment.

---

**Finding SA-002 | P1 | High | Confidence: 0.95**  
**20 reference entities are scoped to `customer` SA — should be in `silver.reference`**

The canonical SA for reference/lookup data is `silver.reference` (SA #1, Priority 1 Critical). The following 20 reference entities are incorrectly scoped to `silver.customer`:

`Reference_SIC_Code`, `Reference_SIC_Group`, `Reference_SIC_EconomicMeasure`, `Reference_Business_SIC_History`, `Reference_Customer_Status`, `Reference_Customer_Type`, `Reference_Country`, `Reference_Ethnicity`, `Reference_Gender`, `Reference_Organization_Type`, `Reference_ID_Type`, `Reference_Education`, `Reference_Own_Home`, `Reference_Marital_Status`, `Reference_Occupation_PermanentResident`, `Reference_Residence_Type`, `Reference_Segment`, `Reference_Customer_Business_Classification`, `Reference_Relationship_Manager`, `Reference_Credit_Grade`.

Of these, only 3 have physical tables (`reference_country`, `reference_customer_type`, `reference_segment`). The remaining 17 are unmaterialized.

**Remediation**: Migrate physical tables to `silver.reference` schema. Redirect foreign key lookups. Remove from customer SA scope.

---

**Finding SA-003 | P1 | Medium | Confidence: 0.90**  
**`credit_limit_liability` physical table has no corresponding logical entity**

The physical table `silver_dev_v2.customer.credit_limit_liability` exists in the Unity Catalog but has no corresponding entity in the logical model and no mapping in the data mapping document. This is either:
(a) A business object that was deployed without a logical model (governance gap — guideline §2.11: "Tables that exist in the platform but have no corresponding entity in the Erwin model are a governance gap"), or  
(b) Data that belongs to the `silver.account` or `silver.lending` subject area.

**Remediation**: Raise with data steward. If valid, model the entity in the logical model. If misplaced, migrate to correct SA.

---

### 2.2 Identity Strategy

**Finding ID-001 | P0 | High | Confidence: 1.00**  
**SLV-001 VIOLATED: No `<entity>_sk` (MD5 surrogate key) on any entity**

*Guideline §2.4 (Modeling DOs & DON'Ts): "Derive surrogate keys deterministically: `MD5(UPPER(TRIM(business_key)))` for single-component keys."*  
*Rule SLV-001 per assessment framework: `<entity>_sk` (MD5) on every entity.*

A search of all 584 logical model attributes for patterns containing `_sk` returns **zero matches**. No entity in the customer SA logical model defines a surrogate key. This means:
- Downstream Gold joins using deterministic SK-based SCD-2 patterns cannot be built.
- Pipeline idempotency cannot be guaranteed (rebuilds may generate orphaned records).
- FK relationships across Silver entities are undefined.

Affects all 11 core entities and all 20 reference entities.

---

**Finding ID-002 | P0 | High | Confidence: 1.00**  
**SLV-002 VIOLATED: No `<entity>_bk` (business key) on any entity**

*Rule SLV-002: `<entity>_bk` on every entity.*

A search of all 584 logical model attributes for patterns containing `_bk` returns **zero matches**. The "Customer number (CIF)" attribute in the `Customer` entity is semantically the business key but:
- Uses a descriptive label with spaces, parentheses — not `customer_bk` (NAM-001 violation).
- Is typed as `STRING` (no further precision).
- Does not appear on child entities (`Customer_ContactInformation`, `Customer_Demographics`, etc.) as a named FK — these entities carry "Customer number (CIF)" and "Customer Type" as composite PK candidates but without explicit FK naming convention.

**Remediation**: Rename to `customer_bk` on parent entity. Add `customer_bk` as FK on all child entities. Add `<entity>_sk = MD5(UPPER(TRIM(customer_bk)))` on all entities.

---

### 2.3 SCD Strategy

**Finding SCD-001 | P0 | High | Confidence: 1.00**  
**SLV-003 VIOLATED: SCD-2 columns incorrectly named, typed, and incomplete**

*Guideline §2.3 (Modeling DOs & DON'Ts): "Every Silver entity table must have `effective_from`, `effective_to`, `is_current`."*  
*Guideline §5 (DQ Controls): Mandatory audit columns include `is_current`, `valid_from_at`, `valid_to_at`.*

Current state across all core entities:

| Required Column | Actual Column in LM | Type in LM | Issues |
|---|---|---|---|
| `effective_from` / `valid_from_at` | `Business Start Date` | `CHAR(18)` | Wrong name; wrong type (must be TIMESTAMP) |
| `effective_to` / `valid_to_at` | `Business End Date` | `CHAR(18)` | Wrong name; wrong type (must be TIMESTAMP) |
| `is_current` | `IsLogicallyDeleted` | `STRING` | Wrong name; inverted semantics (deleted ≠ current); camelCase violates NAM-001 |
| `run_id` | *(absent)* | — | Missing from all entities |
| `source_ingestion_date` | *(absent)* | — | Missing from all entities |

Additionally, the `appendOnly` Delta feature is enabled on the following tables, which **prevents** the `DELETE`/`UPDATE` operations required by SCD-2 MERGE patterns: `reference_country`, `reference_customer_type`, `customer_documents`, `customer_preferences`, `customer_contactinformation`, `customer`, `credit_limit_liability`, `customer_demographics`, `customer_relationships`.

**Remediation**: Rename `Business Start Date` → `effective_from` (TIMESTAMP NOT NULL). Rename `Business End Date` → `effective_to` (TIMESTAMP). Rename `IsLogicallyDeleted` → `is_current` (BOOLEAN NOT NULL). Add `run_id` (STRING) and `source_ingestion_date` (TIMESTAMP). Remove `appendOnly` constraint from all SCD-managed tables.

---

**Finding SCD-002 | P1 | Medium | Confidence: 0.85**  
**SCD type assignment undocumented for all entities**

There is no documentation (in the logical model, mapping, or guidelines) specifying which entities use SCD-1 vs SCD-2. Reference entities (`reference_country`, `reference_segment`) likely require SCD-1 (dimension tables that are overwritten). Core customer entities require SCD-2. No steward approval for SCD-1 exceptions is documented.

**Guideline**: *"SCD-1 requires documented data-steward approval"* (guideline §2.3, SLV-003).

---

### 2.4 Cross-Entity Referential Integrity

**Finding RI-001 | P0 | High | Confidence: 1.00**  
**No foreign keys defined between any entities**

The logical model defines no explicit FK relationships. All child entities (`Customer_ContactInformation`, `Customer_Demographics`, `Customer_Documents`, `Customer_Groupings`, `Customer_Preferences`, `Customer_Psychographics`, `Customer_Relationships`) carry "Customer number (CIF)" and "Customer Type" as composite natural key references but:
- These are not named `customer_sk` or `customer_bk` FK columns.
- No referential integrity is enforced.
- No join path is formally documented.

**DQ Controls §3.1**: *"Referential Integrity: All foreign keys must reference valid dimension records."*

---

**Finding RI-002 | P1 | High | Confidence: 0.90**  
**`Customer_Relationships` entity references related CIF IDs with no self-referencing FK**

The `Customer_Relationships` entity contains `CIF ID`, `Related CIF ID`, and `CIF Type` attributes. These imply a self-join to the `Customer` entity. No FK is defined. The "Relationship Type" attribute is unmapped (status: N), preventing classification of relationship semantics (guarantor, UBO, household member, etc.).

---

### 2.5 ETL Mapping & Lineage

**Finding ETL-001 | P0 | High | Confidence: 1.00**  
**Overall mapping completeness is 41% — majority of the SA is unmapped**

| Entity | LM Attributes | Mapping Rows | Mapped (Y) | Unmapped (N) | Coverage |
|---|---|---|---|---|---|
| Customer | 145 | 135 | 55 | 80 | **41%** |
| Customer_AdditionalFinancialInformation | 25 | 14 | 3 | 11 | **21%** |
| Customer_ContactInformation | 30 | 22 | 15 | 7 | **68%** |
| Customer_Demographics | 36 | 27 | 8 | 19 | **30%** |
| Customer_Documents | 28 | 17 | 12 | 5 | **71%** |
| Customer_Groupings | 16 | 5 | 4 | 1 | **80%** |
| Customer_Preferences | 51 | 40 | 3 | 37 | **8%** |
| Customer_Psychographics | 26 | 22 | 14 | 8 | **64%** |
| Customer_Relationships | 40 | 32 | 9 | 23 | **28%** |
| Customer_Individual | 5 | 0 | 0 | 0 | **0%** |
| Customer_Business | 5 | 0 | 0 | 0 | **0%** |
| **Total Core** | **407** | **314** | **123** | **191** | **~39%** |

Additionally, 10 attributes in the Customer entity appear in the LM but not in the mapping document at all (status: `NOT_IN_MAP`), including the critical metadata attributes `Source System Code`, `Source System Id`, `IsLogicallyDeleted`, `Create Date`, `Update Date`, `Delete Date`.

The mapping document contains data quality issues of its own: column header typo `Enitity_Name` (should be `Entity_Name`), entity name typo `Customer_Relatioships` (should be `Customer_Relationships`), column header typo `Souce_System.Column_Data_Type`. These indicate low documentation maturity.

---

**Finding ETL-002 | P0 | High | Confidence: 0.90**  
**Source code canonicalization not applied — raw Finacle codes mapped straight-through**

*Guideline §2.5 (Modeling DOs & DON'Ts): "DO: Resolve all source-system codes to canonical platform values using a central code-mapping reference (e.g., `silver.reference.code_mapping`). DON'T: Store source-system-specific codes directly in Silver entity columns."*  
*Rule SLV-004: No source-system codes; use `silver.reference.code_mapping`.*

The mapping document shows that fields like `Customer Type` (from `CRMUSER.ACCOUNTS.CUST_TYPE`) and `Industry Type` (SIC codes) are mapped as "straight through field" with no transformation to canonical codes. Raw Finacle source codes (e.g., `CUST_TYPE` values from Finacle/CRMUSER) will be stored directly in Silver without canonicalization.

---

**Finding ETL-003 | P1 | Medium | Confidence: 0.85**  
**Multiple source systems (Finacle + RLS Loan) merge logic not documented**

The `Fields Classification` sheet shows several attributes with `Source of change: Multiple` sourced from `Finacle/RLS`. No conflict resolution or precedence logic is documented for cases where both systems provide the same attribute (e.g., `CIF Creation Date`, `Customer Full Name`, `Last Name`, `Domicile Branch`). Guideline §2.10 requires companion tables per source system; instead, a single wide entity is planned.

---

### 2.6 Storage & Physical Infrastructure

**Finding INF-001 | P1 | Medium | Confidence: 1.00**  
**No partitioning on any of the 10 physical tables**

All 10 physical tables have `partitionColumns: []`. For high-volume tables like `customer` (244 MB) and `customer_demographics` (106 MB), absence of partition columns means full table scans on every incremental read. Guideline §1.4 (Modeling DOs & DON'Ts): *"Partition all [tables] by ingestion date (`_ingested_date`)."* At Silver, common partition candidates are `_ingested_date` or `effective_from` date part.

---

**Finding INF-002 | P1 | Medium | Confidence: 1.00**  
**Auto-optimize missing on 7 of 10 tables**

Tables missing both `delta.autoOptimize.autoCompact` and `delta.autoOptimize.optimizeWrite`:

| Table | autoCompact | optimizeWrite |
|---|---|---|
| `reference_country` | ❌ MISSING | ❌ MISSING |
| `reference_customer_type` | ❌ MISSING | ❌ MISSING |
| `customer_contactinformation` | ❌ MISSING | ❌ MISSING |
| `reference_segment` | ❌ MISSING | ❌ MISSING |
| `credit_limit_liability` | ❌ MISSING | ❌ MISSING |
| `customer_demographics` | ❌ MISSING | ❌ MISSING |
| `customer_relationships` | ❌ MISSING | ❌ MISSING |

Only `customer`, `customer_documents`, and `customer_preferences` have auto-optimize enabled.

---

**Finding INF-003 | P2 | Low | Confidence: 1.00**  
**5 tables are empty (0 bytes, 0 files)**

`customer_preferences` (0 bytes), `customer_contactinformation` (0 bytes), `reference_segment` (0 bytes), `credit_limit_liability` (0 bytes), `customer_relationships` (0 bytes). These tables exist in the catalog but have never been loaded. This indicates pipeline implementation is significantly behind model design.

---

**Finding INF-004 | P1 | Medium | Confidence: 0.95**  
**`appendOnly` Delta feature conflicts with SCD-2 MERGE requirement**

Nine tables have the `appendOnly` Delta table feature enabled. This feature disallows `DELETE` and `UPDATE` operations. SCD-2 pattern requires a `MERGE` that updates the prior row's `effective_to` / `is_current`. The `appendOnly` constraint makes SCD-2 physically impossible on these tables without removing the feature first.

---

## 3. Entity Inventory

| # | Entity | Physical Table | Physical Mapped | Schema | SCD Cols Present | SK Present | BK Present | Metadata Complete | P0 Findings |
|---|---|---|---|---|---|---|---|---|---|
| 1 | Customer | `customer` | ✅ | silver_dev_v2.customer | ❌ Wrong names/types | ❌ | ❌ | ❌ | 8 |
| 2 | Customer_AdditionalFinancialInformation | *(none)* | ❌ | — | ❌ | ❌ | ❌ | ❌ | 3 |
| 3 | Customer_ContactInformation | `customer_contactinformation` | ✅ (empty) | silver_dev_v2.customer | ❌ | ❌ | ❌ | ❌ | 4 |
| 4 | Customer_Demographics | `customer_demographics` | ✅ | silver_dev_v2.customer | ❌ | ❌ | ❌ | ❌ | 4 |
| 5 | Customer_Documents | `customer_documents` | ✅ | silver_dev_v2.customer | ❌ | ❌ | ❌ | ❌ | 3 |
| 6 | Customer_Groupings | *(none)* | ❌ | — | ❌ | ❌ | ❌ | ❌ | 2 |
| 7 | Customer_Preferences | `customer_preferences` | ✅ (empty) | silver_dev_v2.customer | ❌ | ❌ | ❌ | ❌ | 3 |
| 8 | Customer_Psychographics | *(none)* | ❌ | — | ❌ | ❌ | ❌ | ❌ | 2 |
| 9 | Customer_Relationships | `customer_relationships` | ✅ (empty) | silver_dev_v2.customer | ❌ | ❌ | ❌ | ❌ | 3 |
| 10 | Customer_Individual | *(none)* | ❌ | — | ❌ | ❌ | ❌ | ❌ | 2 |
| 11 | Customer_Business | *(none)* | ❌ | — | ❌ | ❌ | ❌ | ❌ | 2 |
| 12–31 | Reference_* (20 entities) | 3 of 20 physical | Partial | silver_dev_v2.customer ❌ wrong SA | ❌ | ❌ | ❌ | ❌ | 1 each |

---

## 4. Per-Entity Assessments

### 4.1 Customer

#### 4.1.1 Industry Fitness

**Boundary / Grain**: The `Customer` entity intends to represent the single root entity for all customer master data. The grain appears to be one row per customer CIF (Customer Information File), with SCD-2 tracking changes over time. However, the entity suffers from severe scope creep, mixing:
- **Party identity attributes** (Full Name, Title, DoB, Nationality, EID) → belong in a `Party` entity per BIAN/IFW
- **Customer role attributes** (Customer Segment Code, VIP Indicator, Primary RM Code) → valid for a `CustomerRole` entity
- **Product relationship attributes** (Available Products, Selected Products) → should be in product or account SA
- **AML/risk attributes** (Risk Profile, Credit Grading, Watchlist) → should be in `silver.risk` SA
- **Banking channel preferences** (Enable E-Banking, SMS Banking, WAP Banking) → should be in `Customer_Preferences` entity
- **Relationship Manager contact details** (Primary/Secondary/Tertiary RM Name/Mobile/Email) → denormalization of RM entity (repeating group violation)
- **SIC classification** (Industry Type, Sector, Sub Sector) → denormalized from reference entities

**BIAN Alignment**: The entity conflates BIAN *Party Data Management* domain and BIAN *Customer Management* service domain. These should be separated per IFW design.

**3NF Violation**: Multiple repeating groups exist:
1. Three RM tiers (Primary/Secondary/Tertiary RM Code/Name/Mobile/Email) — 12 attributes forming a repeating group; should be a child `customer_relationship_manager` table.
2. Two call-back contacts (Primary/Secondary Call Back Contact/Number) — 4 attributes forming a repeating group.
3. Consent flags (multiple `Do Not Call/SMS/Mail/Email` indicators) — overlaps `Customer_Preferences` entity, creating duplication.

**Missing Attributes**: `customer_sk`, `customer_bk`, `record_hash`, `batch_id`, `data_quality_status`, `dq_issues`.

#### 4.1.2 Attribute-Level Review — Customer Entity (Selected Critical Attributes)

| Attribute | Physical Col Candidate | LM Type | Issues |
|---|---|---|---|
| Customer number (CIF) | `customer_bk` (should be renamed) | STRING | Not named `customer_bk`; NAM-001 violation (spaces, parens) |
| *(missing)* | `customer_sk` | — | **MISSING** — no surrogate key defined at all |
| Customer Type | Raw from CRMUSER.ACCOUNTS.CUST_TYPE | STRING | SLV-004 violation: raw source code, no canonicalization |
| CIF Creation Date | TMDATE | STRING | Wrong type: should be DATE; date stored as STRING |
| Customer Full Name | CUST_FIRST_NAME \|\| CUST_MIDDLE_NAME \|\| CUST_LAST_NAME | STRING | Derived/concatenated value at Silver violates SLV-006 |
| Birth date | — | STRING | Wrong type: should be DATE |
| Gross Monthly Income | — | STRING | Wrong type: should be DECIMAL(18,4) or BIGINT (fils); SLV-010 violation |
| VIP Indicator | — | STRING | Should be BOOLEAN; `is_vip` naming convention (NAM-001) |
| Suspended | — | STRING | Should be BOOLEAN; `is_suspended` per NAM-001 |
| Blacklisted | — | STRING | Should be BOOLEAN; `is_blacklisted` per NAM-001 |
| Minor Indicator | — | STRING | Should be BOOLEAN; `is_minor` per NAM-001 |
| Non Resident Indicator | — | STRING | Should be BOOLEAN; `is_non_resident` per NAM-001 |
| Staff indicator | — | STRING | Should be BOOLEAN; `is_staff` per NAM-001 |
| Dormant | — | STRING | Should be BOOLEAN; `is_dormant` per NAM-001 |
| Business Start Date | — | CHAR(18) | Should be TIMESTAMP named `effective_from`; SCD-001 |
| Business End Date | — | CHAR(18) | Should be TIMESTAMP named `effective_to`; SCD-001 |
| IsLogicallyDeleted | — | STRING | Wrong name (camelCase + wrong semantics); should be `is_current BOOLEAN` |
| Source System Code | — | STRING | Not mapped (??); required audit column |
| Source System Id | — | STRING | Not mapped (??); required audit column |
| Create Date | — | STRING | Not mapped (??); should be TIMESTAMP |
| Update Date | — | STRING | Not mapped (??); should be TIMESTAMP |
| Delete Date | — | STRING | Not mapped (??); should be TIMESTAMP |
| *(missing)* | `run_id` | — | **MISSING** from LM entirely |
| *(missing)* | `source_ingestion_date` | — | **MISSING** from LM entirely |
| Primary RM Code | — | STRING | Mapped (Y) — but starts a repeating group with 11 similar attributes |
| Secondary RM Code | — | STRING | Repeating group: RM tier denormalization |
| Tertiary RM Code | — | STRING | Repeating group: RM tier denormalization |
| Primary RM Name | — | STRING | Unmapped (N); repeating group |
| Primary RM Mobile | — | STRING | Unmapped (N); PII field; needs SHA-256 masking per DQ §2 |
| Primary RM Email | — | STRING | Unmapped (N); PII; repeating group |
| Credit Bureau Score | — | STRING | Wrong type: should be INTEGER or DECIMAL; unmapped |
| Moody's Rating | — | STRING | Belongs in `silver.risk` SA |
| Available Products | — | STRING | Belongs in `silver.product`/`silver.account` SA |
| Enable E-Banking | — | STRING | Should be BOOLEAN; channel preference → should be in Customer_Preferences |
| Customer Full Name | — | STRING | "Customer Full Name" stored as concatenation of 3 source fields (Finacle col: `CUST_FIRST_NAME \|\| CUST_MIDDLE_NAME \|\| CUST_LAST_NAME`) — derived value violates SLV-006 |

**Summary**: 55/145 attributes mapped (38%). 80 unmapped. 10 not in mapping document at all. All attributes STRING-typed, none use domain-appropriate types.

#### 4.1.3 Metadata Completeness — Customer

| Metadata Column | In LM? | Correctly Named? | Correct Type? | Mapped? | Finding |
|---|---|---|---|---|---|
| `source_system_code` | ✅ (as "Source System Code") | ❌ Title Case, spaces | ❌ STRING (ok) | ❌ NOT_IN_MAP | Rename to `source_system_code` |
| `source_system_id` | ✅ (as "Source System Id") | ❌ Title Case | ❌ STRING | ❌ NOT_IN_MAP | Rename |
| `create_date` | ✅ (as "Create Date") | ❌ Title Case | ❌ STRING → TIMESTAMP | ❌ NOT_IN_MAP | Fix name + type |
| `update_date` | ✅ (as "Update Date") | ❌ Title Case | ❌ STRING → TIMESTAMP | ❌ NOT_IN_MAP | Fix name + type |
| `delete_date` | ✅ (as "Delete Date") | ❌ Title Case | ❌ STRING → TIMESTAMP | ❌ NOT_IN_MAP | Fix name + type |
| `is_active_flag` | ✅ (as "Is Active Flag") | ❌ Title Case | ❌ CHAR(18) → STRING | ❌ NOT_IN_MAP | Fix name + type |
| `effective_from` | ✅ (as "Business Start Date") | ❌ Wrong name | ❌ CHAR(18) → TIMESTAMP | ❌ NOT_IN_MAP | Critical: rename + retype |
| `effective_to` | ✅ (as "Business End Date") | ❌ Wrong name | ❌ CHAR(18) → TIMESTAMP | ❌ NOT_IN_MAP | Critical: rename + retype |
| `is_current` | ✅ (as "IsLogicallyDeleted") | ❌ Wrong name + camelCase | ❌ STRING → BOOLEAN | ❌ NOT_IN_MAP | Critical: wrong semantics |
| `run_id` | ❌ ABSENT | — | — | — | **MISSING** from LM |
| `source_ingestion_date` | ❌ ABSENT | — | — | — | **MISSING** from LM |
| `record_source` (DQ §5) | ❌ ABSENT | — | — | — | **MISSING** — DQ Controls §5 mandatory |
| `record_origin` (DQ §5) | ❌ ABSENT | — | — | — | **MISSING** |
| `record_hash` (DQ §5) | ❌ ABSENT | — | — | — | **MISSING** |
| `batch_id` (DQ §5) | ❌ ABSENT | — | — | — | **MISSING** |
| `data_quality_status` (DQ §5) | ❌ ABSENT | — | — | — | **MISSING** |
| `dq_issues` (DQ §5) | ❌ ABSENT | — | — | — | **MISSING** |

**Result: 0 of 17 required metadata columns fully compliant.**

---

### 4.2 Customer_AdditionalFinancialInformation

#### 4.2.1 Industry Fitness

**Boundary**: Models financial profile data for a customer: net worth, income, borrowing history, and external banking relationships. This entity mixes:
- **AML/risk profile data** (Expatriate Risk, Overseas Risk, IAS Standards, Credit Policy Breach) → belongs in `silver.risk` SA
- **External bank relationship data** (Bank Name, Branch Name, Product Category, Account ID) → belongs in relationship or account SA

**BIAN Alignment**: Maps loosely to BIAN *Customer Relationship Management* and *Credit Management*. The AML attributes should be in BIAN *Party Authentication/Party Reference Data* within `silver.risk`.

**Physical**: **No physical table exists.** All attributes are unmaterialized.

#### 4.2.2 Attribute-Level Review

| Attribute | LM Type | Map Status | Finding |
|---|---|---|---|
| Customer number (CIF) | STRING | NOT_IN_MAP | Missing FK to parent Customer entity; should be `customer_bk` FK |
| Customer Type | STRING | NOT_IN_MAP | Repeating PK pattern without SK |
| Net worth | STRING | N | Wrong type: should be DECIMAL(18,4); SLV-010 (monetary amount) |
| Expatriate Risk | STRING | N | Should be in `silver.risk` SA; BOOLEAN flag |
| Overseas Risk | STRING | N | Should be in `silver.risk` SA |
| IAS Standards | STRING | N | Ambiguous definition; belongs in risk SA |
| Credit Policy Breach | STRING | N | Should be BOOLEAN; belongs in `silver.risk` |
| Borrowing Start Date | STRING | N | Should be DATE type |
| Name of Auditor | STRING | NOT_IN_MAP | PII-adjacent; external party reference |
| Audited Financials Held | STRING | N | Should be BOOLEAN (`is_audited_financials_held`) |
| Average Annual Income | STRING | N | Wrong type: DECIMAL(18,4) or BIGINT (fils); SLV-010 |
| Bank Name | STRING | Y | External bank; consider normalization |
| Branch Name | STRING | Y | Denormalized bank branch |
| Product Category | STRING | Y | Ambiguous — which product catalog? |
| A/c. ID | STRING | N | Poorly named: `account_id` per NAM-001 |
| Deposits/Inv/Savings | STRING | N | Aggregated value: violates SLV-007 (no pre-computed metrics); also wrong type |
| Source System Code | STRING | NOT_IN_MAP | Technical audit column |
| Source System Id | STRING | NOT_IN_MAP | Technical audit column |
| IsLogicallyDeleted | STRING | NOT_IN_MAP | Wrong name/semantics; should be `is_current` BOOLEAN |
| Create Date | STRING | NOT_IN_MAP | Should be TIMESTAMP |
| Update Date | STRING | NOT_IN_MAP | Should be TIMESTAMP |
| Delete Date | STRING | NOT_IN_MAP | Should be TIMESTAMP |
| Business Start Date | CHAR(18) | NOT_IN_MAP | Should be TIMESTAMP named `effective_from` |
| Business End Date | CHAR(18) | NOT_IN_MAP | Should be TIMESTAMP named `effective_to` |
| Is Active Flag | CHAR(18) | NOT_IN_MAP | Should be STRING/BOOLEAN |

**Summary**: 3/25 mapped (12%). Entity not materialized. Contains misplaced risk attributes.

#### 4.2.3 Metadata Completeness
All 17 required metadata columns missing or non-compliant. Same pattern as §4.1.3. Physical table absent.

---

### 4.3 Customer_ContactInformation

#### 4.3.1 Industry Fitness

**Boundary**: Models customer contact data — addresses, phone, email, preferred contact. **Critical 3NF violation**: The entity uses numbered address columns (`Mail address1–4`, `Registered Address1–4`) which are repeating groups violating 1NF/3NF.

*Guideline §2.1 (Modeling DOs & DON'Ts): "DO NOT add numbered or suffixed columns to represent multiple instances of the same concept (e.g., `address1`, `address2`, `address3`, `address4`)."*

The correct pattern is a separate normalized `customer_address` child table with `address_type_code`, `address_line_1`, `address_line_2`, `city`, `emirate_code`, `postal_code`, `country_code`, `is_primary` — exactly as shown in the guideline's worked example.

Similarly, `Phone No. / E-mail` is a single column trying to contain two different contact channel types — a data type and semantic violation.

**Physical**: Physical table `customer_contactinformation` exists but is **empty** (0 bytes, 0 files). Note: physical table name should be `customer_contact_information` (underscore between words per NAM-001).

#### 4.3.2 Attribute-Level Review

| Attribute | LM Type | Map Status | Finding |
|---|---|---|---|
| Customer number (CIF) | STRING | NOT_IN_MAP | FK missing; not `customer_bk` |
| Customer Type | STRING | NOT_IN_MAP | PK composite without SK |
| E-statement delivery indicator | STRING | N | Should be BOOLEAN `is_estatement_enabled` |
| Mail address1 | STRING | Y | **3NF violation**: numbered repeating group |
| Mail address2 | STRING | Y | **3NF violation** |
| Mail address3 | STRING | Y | **3NF violation** |
| Mail address4 | STRING | N | **3NF violation** |
| Registered Address1 | STRING | Y | **3NF violation** |
| Registered Address2 | STRING | Y | **3NF violation** |
| Registered Address3 | STRING | Y | **3NF violation** |
| Registered Address4 | STRING | N | **3NF violation** |
| Phone number | STRING | N | PII: requires SHA-256 masking per DQ §2 |
| Email id | STRING | N | PII: SHA-256 masking; `email_id` → `email_address` per naming |
| Mobile Number | STRING | Y | PII: SHA-256 masking required |
| Postal code | STRING | Y | OK |
| Preferred address based on CFMAST | STRING | Y | Non-standard naming; source system reference leaking into Silver |
| Preferred address based on CMAST | STRING | Y | Source system reference leaking |
| Preferred Flag | STRING | Y | Should be BOOLEAN `is_preferred` |
| Preferred Address Type | STRING | Y | Should be code value canonicalized via `silver.reference.code_mapping` |
| Country | STRING | Y | Should be ISO 3166-1 alpha-3 `country_code CHAR(3)` |
| Business Center Name | STRING | Y | Denormalized; FK to organization SA preferred |
| Phone No. / E-mail | STRING | Y | Invalid multi-type column; split into separate columns |
| Preferred Contact No. Type | STRING | N | Should be canonicalized code |
| Preferred Email ID Type | STRING | N | Should be canonicalized code |

**Summary**: 15/30 mapped (50%). All 8 address-line columns violate 3NF. Empty physical table.

#### 4.3.3 Metadata Completeness
Same deficiencies as §4.1.3. Physical table empty — no data loaded.

---

### 4.4 Customer_Demographics

#### 4.4.1 Industry Fitness

**Boundary**: Models individual customer demographic profile: nationality, income, employment, marital status, education. Generally correct scope, though:
- `US Relation` and `TIN Number` are FATCA/CRS compliance fields → arguable overlap with `silver.risk` SA.
- `Monthly Income` and `Annual income` are both present — potential duplicate of `Gross Monthly Income` in the parent `Customer` entity (denormalization).

**Physical**: Physical table `customer_demographics` exists with data (106 MB, 3 files). One of only 3 tables with data.

#### 4.4.2 Attribute-Level Review

| Attribute | LM Type | Map Status | Finding |
|---|---|---|---|
| Customer number (CIF) | STRING | NOT_IN_MAP | FK missing |
| Customer Type | STRING | NOT_IN_MAP | PK without SK |
| US Relation | STRING | N | FATCA field: `is_us_related` BOOLEAN; consider risk SA |
| TIN Number | STRING | Y | PII: SHA-256 masking; FATCA field |
| Monthly Income | STRING | Y | Wrong type: DECIMAL(18,4)/BIGINT; potential duplicate of `Gross Monthly Income` in Customer |
| Annual income | STRING | N | Wrong type: DECIMAL(18,4)/BIGINT; potential duplicate |
| Gender code | STRING | Y | Should be canonicalized via `silver.reference.code_mapping` |
| Marital status | STRING | Y | Should be canonicalized code |
| Occupation code permanent resident indicator | STRING | Y | Poor naming: not snake_case; overly long |
| Occupation | STRING | Y | Should be canonicalized SIC/occupation code |
| Job designation | STRING | Y | Free text: consider code canonicalization |
| Current employer name | STRING | Y | PII-adjacent; free text |
| Monthly Income | STRING | Y | Type: DECIMAL(18,4) required |
| Number of dependents | STRING | N | Should be INTEGER |
| Education code | STRING | N | Should be canonicalized via `silver.reference.code_mapping` |
| Ethnic code | STRING | N | Sensitive PII; should be access-controlled |
| Expiry Date | STRING | N | Should be DATE; expiry of what? — ambiguous definition |
| Current employment | STRING | N | Should be BOOLEAN `is_employed` |
| Current employment tenure | STRING | N | Should be INTEGER (months) |
| Employed Date | STRING | N | Should be DATE |
| Place of Birth | STRING | N | Should be normalized: `city_of_birth`, `country_of_birth` |
| Country of Birth | STRING | N | Should be country_code CHAR(3) |
| Country of Tax Residence | STRING | N | Should be country_code CHAR(3) |
| Residing Country | STRING | N | Duplicate of `Country of domicile` in Customer entity |
| Residence type | STRING | N | Canonicalize via reference |
| Own home Indicator | STRING | N | `is_own_home` BOOLEAN |
| Signed Date | STRING | N | Should be DATE; context missing |
| Reason Or Type of Relation | STRING | N | Ambiguous; US relation context |
| Source System Code | STRING | NOT_IN_MAP | Audit column |
| Source System Id | STRING | NOT_IN_MAP | Audit column |
| IsLogicallyDeleted | STRING | NOT_IN_MAP | Wrong column |
| Create Date | STRING | NOT_IN_MAP | TIMESTAMP |
| Update Date | STRING | NOT_IN_MAP | TIMESTAMP |
| Delete Date | STRING | NOT_IN_MAP | TIMESTAMP |
| Business Start Date | CHAR(18) | NOT_IN_MAP | TIMESTAMP `effective_from` |
| Business End Date | CHAR(18) | NOT_IN_MAP | TIMESTAMP `effective_to` |
| Is Active Flag | CHAR(18) | NOT_IN_MAP | Fix type |

**Summary**: 8/36 mapped (22%). Multiple monetary/boolean/date type violations. Income duplicated from Customer entity.

#### 4.4.3 Metadata Completeness
Same deficiencies as §4.1.3.

---

### 4.5 Customer_Documents

#### 4.5.1 Industry Fitness

**Boundary**: Models identity documents — trade licenses, EID, passport, CRS flags, and general document registry. Generally well-scoped. However:
- `Passport Number` and `EID Number` are critical PII and must be SHA-256 masked per DQ Controls §2: *"EID, Card Numbers, Phone Numbers, and Passport Numbers masked via SHA-256 unless stored in a restricted container."* Both are currently unmapped (status: N), but when mapped must be masked.
- Both `Passport Number` (from `Customer_Documents`) and `EID Number` are also referenced in the `Customer` entity logical model (`ID number`, `ID type`) — potential duplication.

**Physical**: Physical table `customer_documents` exists with data (35 MB, 1 file).

#### 4.5.2 Attribute-Level Review

| Attribute | LM Type | Map Status | Finding |
|---|---|---|---|
| Customer number (CIF) | STRING | NOT_IN_MAP | FK missing |
| Customer Type | STRING | NOT_IN_MAP | PK without SK |
| Passport Number | STRING | N | **PII — SHA-256 masking mandatory** per DQ Controls §2 |
| EID Number | STRING | N | **PII — SHA-256 masking mandatory** per DQ Controls §2 |
| Trade License Number | STRING | Y | PII-adjacent; masking TBD |
| Documents Collected | STRING | N | Should be BOOLEAN |
| CRS Undocumented Flag | STRING | Y | Should be BOOLEAN |
| CRS Undocumented Flag Reason | STRING | Y | OK |
| Document Code | STRING | Y | Canonicalize via `silver.reference.code_mapping` |
| Document Name | STRING | Y | Derived from Document Code? — check SLV-006 |
| Unique ID | STRING | Y | Ambiguous naming; should be `document_id` per NAM-001 |
| Issue Date | STRING | Y | Wrong type: should be DATE |
| Document Type | STRING | Y | Canonicalize |
| Document Type Description | STRING | Y | Derived from code — SLV-006 risk |
| Document Code Description | STRING | Y | Derived from Document Code — SLV-006 violation |
| Trade License Source | STRING | N | Should be canonicalized |
| Trade License Update Status | STRING | N | Should be canonicalized BOOLEAN/code |
| Is Document Verified? | STRING | Y | `is_document_verified` BOOLEAN per NAM-001 (question mark in name violates NAM-001) |
| IDIssued Organization | STRING | Y | `id_issuing_organization` — snake_case required |
| Source System Code | STRING | NOT_IN_MAP | Audit column |
| Source System Id | STRING | NOT_IN_MAP | Audit column |
| IsLogicallyDeleted | STRING | NOT_IN_MAP | Wrong column |
| Create Date | STRING | NOT_IN_MAP | TIMESTAMP |
| Update Date | STRING | NOT_IN_MAP | TIMESTAMP |
| Delete Date | STRING | NOT_IN_MAP | TIMESTAMP |
| Business Start Date | CHAR(18) | NOT_IN_MAP | `effective_from` TIMESTAMP |
| Business End Date | CHAR(18) | NOT_IN_MAP | `effective_to` TIMESTAMP |
| Is Active Flag | CHAR(18) | NOT_IN_MAP | Fix type |

**Summary**: 12/28 mapped (43%). Critical PII masking gap. Naming violations. Description columns risk SLV-006.

#### 4.5.3 Metadata Completeness
Same deficiencies as §4.1.3.

---

### 4.6 Customer_Groupings

#### 4.6.1 Industry Fitness

**Boundary**: Models customer group membership (holding groups, related-party groups). Generally correct scope. `Shareholding Pcnt.` (shareholding percentage) is a percentage metric — per SLV-007 and guideline §2.6, pre-computed metrics/percentages should not be stored in Silver.

**Physical**: **No physical table.** Entity not materialized.

#### 4.6.2 Attribute-Level Review

| Attribute | LM Type | Map Status | Finding |
|---|---|---|---|
| Customer number (CIF) | STRING | NOT_IN_MAP | FK missing |
| Customer Type | STRING | NOT_IN_MAP | PK without SK |
| Group ID | STRING | Y | Should be `group_bk` or `group_id` per NAM guidelines |
| Group Code | STRING | Y | Canonicalize via reference |
| Group Name | STRING | Y | OK |
| Primary Grp Indicator | STRING | Y | `is_primary_group` BOOLEAN; "Grp" is non-approved abbreviation |
| Shareholding Pcnt. | STRING | N | **SLV-007 violation**: percentage metric; wrong type (DECIMAL) |
| Source System Code | STRING | NOT_IN_MAP | Audit column |
| Source System Id | STRING | NOT_IN_MAP | Audit column |
| IsLogicallyDeleted | STRING | NOT_IN_MAP | Wrong |
| Create Date | STRING | NOT_IN_MAP | TIMESTAMP |
| Update Date | STRING | NOT_IN_MAP | TIMESTAMP |
| Delete Date | STRING | NOT_IN_MAP | TIMESTAMP |
| Business Start Date | CHAR(18) | NOT_IN_MAP | `effective_from` TIMESTAMP |
| Business End Date | CHAR(18) | NOT_IN_MAP | `effective_to` TIMESTAMP |
| Is Active Flag | CHAR(18) | NOT_IN_MAP | Fix type |

**Summary**: 4/16 mapped (25%). Entity not materialized. Shareholding metric violates SLV-007.

#### 4.6.3 Metadata Completeness
All required metadata missing. No physical table.

---

### 4.7 Customer_Preferences

#### 4.7.1 Industry Fitness

**Boundary**: Models customer communication preferences (consent flags, Do Not Call/SMS/Email/Mail registries, language preference, FATCA/CRS classification flags). Overly broad: mixes:
- **Communication preferences** (Do Not Call, Preferred Language)
- **FATCA/CRS compliance attributes** (US Relation, TIN Number, GIIN Number, Financial Entity — these are regulatory compliance attributes belonging in `silver.risk`)
- **Trade License waiver flags** (operational attributes that may belong in `silver.account` or a product SA)

**Mapping Completeness**: Only 3/51 (8%) attributes are mapped — the lowest of all entities. This entity is substantially undeveloped.

**Physical**: Physical table `customer_preferences` exists but is **empty** (0 bytes).

#### 4.7.2 Attribute-Level Review

| Attribute | LM Type | Map Status | Finding |
|---|---|---|---|
| Customer number (CIF) | STRING | NOT_IN_MAP | FK missing |
| Customer Type | STRING | NOT_IN_MAP | PK without SK |
| Customer Preferred Language | STRING | N | `preferred_language_code` per NAM-001 |
| Preferred Branch | STRING | N | Should be FK to branch; `preferred_branch_bk` |
| Preferred Mobile Number | STRING | N | PII — SHA-256 masking |
| Preferred Email id | STRING | N | PII — SHA-256 masking; `preferred_email_address` |
| Do Not Call indicator in Bank registry | STRING | N | `is_do_not_call_bank` BOOLEAN; long name |
| Do Not SMS indicator in Bank registry | STRING | N | `is_do_not_sms_bank` BOOLEAN |
| Do Not Mail indicator in Bank registry | STRING | N | `is_do_not_mail_bank` BOOLEAN |
| Do Not Email indicator in Bank registry | STRING | N | `is_do_not_email_bank` BOOLEAN |
| Do Not Call indicator in National registry | STRING | N | `is_do_not_call_national` BOOLEAN |
| Do Not SMS indicator in National registry | STRING | N | BOOLEAN |
| Do Not Fax indicator in National registry Standard Industrial Classification code | STRING | N | **Invalid**: field name conflates two unrelated concepts (fax indicator + SIC code) — data model defect |
| Consent to call Bank | STRING | N | `is_consent_call` BOOLEAN |
| Consent to email Bank | STRING | N | `is_consent_email` BOOLEAN |
| Consent to sms Bank | STRING | N | `is_consent_sms` BOOLEAN |
| Consent to mail Bank | STRING | N | `is_consent_mail` BOOLEAN |
| Preference of Language | STRING | N | Duplicate of `Customer Preferred Language` |
| Trade License Alert Waiver Flag | STRING | N | `is_trade_license_alert_waiver` BOOLEAN |
| Trade License Alert waiver Expiry Date | STRING | N | DATE type |
| Trade License Charge Waiver Flag | STRING | N | BOOLEAN |
| Date for Trade License Charge Collection | STRING | N | DATE type |
| Litigations If any | STRING | N | Should be BOOLEAN `has_litigation`; `if any` violates naming |
| Ledger Fee Waiver Special Condition | STRING | N | Should be in account SA |
| Dealing with Countries | STRING | N | Free text; AML scope — belongs in risk SA |
| Countries Set | STRING | N | AML scope |
| Category | STRING | N | Ambiguous; what category? |
| Criteria for U S Entity | STRING | N | FATCA — belongs in risk SA |
| Exempted U S Entity | STRING | N | FATCA — `is_us_entity_exempt` BOOLEAN; risk SA |
| Documents Available | STRING | Y | Should be BOOLEAN |
| Documents Collected | STRING | N | Duplicate of `Customer_Documents.Documents Collected` |
| TIN Number | STRING | N | PII; FATCA — duplicated in `Customer_Demographics` |
| Type | STRING | N | **Invalid name** — reserved word `type`; NAM-005 violation |
| Fatca Entity Type | STRING | N | Belongs in risk SA; `fatca_entity_type_code` |
| Name of security Market | STRING | N | Belongs in wealth SA |
| Name of Traded corporation | STRING | Y | Belongs in wealth/product SA |
| Financial Entity | STRING | Y | FATCA scope — risk SA |
| GIIN Number | STRING | N | FATCA scope |
| U S Relation | STRING | N | FATCA — duplicate of `US Relation` in Demographics |
| Controlling person have U S Relation | STRING | N | FATCA — risk SA |
| Signed Date | STRING | N | DATE type |
| Expiry Date | STRING | N | DATE type |

**Summary**: 3/51 mapped (6%). Critical naming violations including SQL reserved word `type` (NAM-005). Multiple misplaced FATCA attributes. Duplicate attributes with other entities. Empty physical table.

#### 4.7.3 Metadata Completeness
All required metadata missing. Physical table empty.

---

### 4.8 Customer_Psychographics

#### 4.8.1 Industry Fitness

**Boundary**: Models customer KYC and behavioral financial profile: KYC review dates, expected transaction volumes, PEP status, sanctions declaration. This entity is misnamed — "Psychographics" typically means lifestyle/attitudes/values (marketing analytics). The content is actually **KYC profile data** which belongs in `silver.risk` SA per the SA taxonomy: *"silver.risk: KYC Levels, AML Risk Scores, Watchlist Hits, Customer Risk Ratings, FATCA/CRS flags."*

**BIAN alignment**: Maps directly to BIAN *Party Authentication* and *Customer Relationship Management* (KYC sub-domain). The SA taxonomy designates `silver.risk` for these attributes.

**Physical**: **No physical table.** Entity not materialized.

#### 4.8.2 Attribute-Level Review

| Attribute | LM Type | Map Status | Finding |
|---|---|---|---|
| Customer number (CIF) | STRING | NOT_IN_MAP | FK missing |
| Customer Type | STRING | NOT_IN_MAP | PK without SK |
| Watch list indicator | STRING | NOT_IN_MAP | Should be in `silver.risk`; `is_watchlisted` BOOLEAN |
| KYC Held | STRING | N | `is_kyc_held` BOOLEAN; belongs in risk SA |
| KYC Review Date | STRING | Y | DATE type; belongs in risk SA |
| Expected Monthly Credit Turnover Currency Code | STRING | Y | `expected_monthly_credit_turnover_ccy_code CHAR(3)` |
| Expected Monthly Credit Turnover Amount | STRING | Y | DECIMAL(18,4) or BIGINT (fils) — SLV-010 |
| Expected Monthly Cash Credit Turnover Percentage | STRING | Y | Percentage — SLV-007 violation (pre-computed metric); DECIMAL |
| Expected Monthly Non Cash Credit Turnover Percentage | STRING | Y | SLV-007 violation |
| Expected Highest Cash Credit Transaction Currency Code | STRING | Y | `CHAR(3)` |
| Expected Highest Cash Credit Transaction Amount | STRING | Y | BIGINT (fils) |
| Expected Highest Non Cash Credit Transaction Currency Code | STRING | Y | `CHAR(3)` |
| Expected Highest Non Cash Credit Transaction Amount | STRING | N | BIGINT (fils) |
| PEP | STRING | N | `is_pep` BOOLEAN; belongs in risk SA |
| Sanction Declaration Status | STRING | N | Belongs in risk SA; code value |
| DNFBP Declaration Status | STRING | N | Belongs in risk SA |
| No of Permanent Employees | STRING | N | INTEGER; belongs in `Customer_Business` entity |

**Summary**: 14/26 mapped (54%). Entity misnamed and misscoped — content is KYC/AML, belongs in `silver.risk`. Two percentage attributes violate SLV-007. All monetary amounts typed STRING.

#### 4.8.3 Metadata Completeness
All required metadata missing. No physical table.

---

### 4.9 Customer_Relationships

#### 4.9.1 Industry Fitness

**Boundary**: Models related party relationships (UBOs, guarantors, authorized signatories, controlling persons). Generally appropriate scope. However:
- The entity conflates multiple relationship types (guarantor, UBO, household member, authorized signatory) without a discriminator `relationship_type_code` that is mapped — "Relationship Type" is explicitly unmapped (N).
- `Total Liability Pcnt.` and `Total Equity Pcnt.` are percentage metrics → SLV-007 violation.
- Contact and address data (`Address Type`, `Nationality Code`, `Date of Birth`) creates a mini-party profile within the relationship entity, which should instead reference the `Customer`/`Party` entity by FK.

**Physical**: Physical table `customer_relationships` exists but is **empty** (0 bytes, 0 files).

#### 4.9.2 Attribute-Level Review

| Attribute | LM Type | Map Status | Finding |
|---|---|---|---|
| Customer number (CIF) | STRING | NOT_IN_MAP | Parent FK missing |
| Customer Type | STRING | NOT_IN_MAP | PK without SK |
| Relationship Type | STRING | N | **Critical unmapped**: this is the discriminator; canonicalize via `silver.reference.code_mapping` |
| CIF ID | STRING | N | Poorly named; should be `related_customer_bk` or `related_cif_bk` |
| Name | STRING | N | Generic name — violates NAM; PII |
| CIF Type | STRING | N | `related_customer_type_code`; canonicalize |
| Entity Type | STRING | N | `related_entity_type_code` |
| Type of Company | STRING | N | `company_type_code` |
| Controlling Person Type | STRING | N | `controlling_person_type_code` |
| Related CIF ID | STRING | N | Duplicate of CIF ID? Ambiguous |
| Title | STRING | Y | PII; salutation of related party |
| Full Name | STRING | Y | PII — needs masking consideration |
| Gender | STRING | Y | Canonicalize |
| Passport No./ Trade License | STRING | Y | PII — **SHA-256 masking mandatory** |
| Nationality Code | STRING | Y | `CHAR(3)` ISO |
| Nationality | STRING | Y | Derived from Nationality Code — SLV-006 risk |
| Date of Birth/ Date of Incorporation | STRING | Y | DATE type; mixed semantics in one column |
| Total Liability Pcnt. | STRING | N | **SLV-007 violation**: percentage metric |
| Total Equity Pcnt. | STRING | N | **SLV-007 violation**: percentage metric |
| IB Enabled Flag | STRING | N | `is_ib_enabled` BOOLEAN |
| US Relation | STRING | N | FATCA; belongs in risk SA |
| Address Type | STRING | Y | Code; canonicalize |
| TIN Number | STRING | N | PII; FATCA; triplicate across entities |
| % of controlling interest | STRING | N | **SLV-007**: percentage; DECIMAL type; `controlling_interest_pct` |
| Country of Residence Code | STRING | N | `CHAR(3)` |
| Country of Residence | STRING | N | Derived from code — SLV-006 |
| Place Of Birth | STRING | N | Semi-structured; `city_of_birth` |
| Country Of Birth | STRING | N | `CHAR(3)` |
| CRS Undocumented Flag | STRING | N | BOOLEAN; duplicate with `Customer_Documents` |
| CRS Undocumented Flag Reason | STRING | N | Duplicate with `Customer_Documents` |
| Country of Tax Residence | STRING | Y | `CHAR(3)` |

**Summary**: 9/40 mapped (23%). 3 percentage/metric SLV-007 violations. Multiple PII fields requiring masking. Entity not loaded.

#### 4.9.3 Metadata Completeness
All required metadata missing. Physical table empty.

---

### 4.10 Customer_Individual

#### 4.10.1 Industry Fitness

**Boundary**: A logical sub-type of `Customer` for individual (retail) customers. Per BIAN, this maps to *Party Reference Data — Individual Party*. The entity should carry individual-specific attributes not shared with corporate customers. Currently, the logical model contains only 5 attributes (including the composite PK and 3 technical audit columns), making it a near-empty shell. All business-relevant individual attributes appear to have been absorbed into the parent `Customer` entity — a design issue.

**Physical**: **No physical table.** Entity not materialized. **Zero mapping rows.**

#### 4.10.2 Attribute-Level Review

| Attribute | LM Type | Map Status | Finding |
|---|---|---|---|
| Customer number (CIF) | STRING | NOT_IN_MAP | FK to parent; not named `customer_bk` |
| Customer Type | STRING | NOT_IN_MAP | Sub-type discriminator; should be constrained to 'Individual' |
| Business Start Date | CHAR(18) | NOT_IN_MAP | `effective_from` TIMESTAMP |
| Business End Date | CHAR(18) | NOT_IN_MAP | `effective_to` TIMESTAMP |
| Is Active Flag | CHAR(18) | NOT_IN_MAP | Fix type |

**Summary**: 0/5 mapped (0%). Entity is effectively empty of business content — all individual attributes were absorbed into the parent `Customer` entity. The sub-type pattern is broken.

#### 4.10.3 Metadata Completeness
All required metadata absent. No physical table.

---

### 4.11 Customer_Business

#### 4.11.1 Industry Fitness

**Boundary**: A logical sub-type of `Customer` for business/corporate customers. Same structural issue as `Customer_Individual` — only 5 attributes, all are PK/audit columns. Business-specific attributes (trade license, registered address, SIC codes) are scattered across the parent `Customer` entity, `Customer_ContactInformation`, and `Customer_Groupings`. The sub-type entity carries no discriminating business content.

**Physical**: **No physical table.** Entity not materialized. **Zero mapping rows.**

#### 4.11.2 Attribute-Level Review

| Attribute | LM Type | Map Status | Finding |
|---|---|---|---|
| Customer number (CIF) | STRING | NOT_IN_MAP | FK to parent |
| Customer Type | STRING | NOT_IN_MAP | Sub-type discriminator; constrained to corporate/business types |
| Business Start Date | CHAR(18) | NOT_IN_MAP | `effective_from` TIMESTAMP |
| Business End Date | CHAR(18) | NOT_IN_MAP | `effective_to` TIMESTAMP |
| Is Active Flag | CHAR(18) | NOT_IN_MAP | Fix type |

**Summary**: 0/5 mapped (0%). Entity is an empty shell. No physical table.

#### 4.11.3 Metadata Completeness
All required metadata absent. No physical table.

---

### 4.12 Reference Entities (20 entities — bulk assessment)

All 20 reference entities share identical structural deficiencies. Individual assessment is provided below.

#### Common Deficiencies Across All Reference Entities

| Deficiency | Detail |
|---|---|
| **Wrong SA** | All 20 belong in `silver.reference`, not `silver.customer` (SA-002) |
| **No SK/BK** | Zero `<entity>_sk` or `<entity>_bk` attributes in any reference entity |
| **Metadata absent** | No `effective_from`, `effective_to`, `is_current`, `run_id`, `source_ingestion_date` |
| **STRING types** | All reference code values typed STRING; should use domain-appropriate types |
| **Missing physical tables** | 17 of 20 have no physical table |

#### Individual Reference Entity Summary

| Entity | LM Attrs | Mapped | Physical Table | Key Gap |
|---|---|---|---|---|
| Reference_SIC_Code | 9 | 0% | ❌ None | 0% mapped; should be in `silver.reference` |
| Reference_SIC_Group | 8 | 0% | ❌ None | 0% mapped |
| Reference_SIC_EconomicMeasure | 11 | 0% | ❌ None | 0% mapped |
| Reference_Business_SIC_History | 11 | 0% | ❌ None | 0% mapped |
| Reference_Customer_Status | 8 | 100% | ❌ None | No physical table |
| Reference_Customer_Type | 8 | 100% | ✅ `reference_customer_type` | Wrong SA; auto-optimize missing |
| Reference_Country | 16 | 40% | ✅ `reference_country` | Wrong SA; 10/16 attrs unmapped; auto-optimize missing |
| Reference_Ethnicity | 9 | 0% | ❌ None | 0% mapped; sensitive data |
| Reference_Gender | 10 | 100% | ❌ None | No physical table |
| Reference_Organization_Type | 8 | 100% | ❌ None | No physical table |
| Reference_ID_Type | 9 | 100% | ❌ None | No physical table |
| Reference_Education | 8 | 100% | ❌ None | No physical table |
| Reference_Own_Home | 8 | 0% | ❌ None | 0% mapped |
| Reference_Marital_Status | 8 | 100% | ❌ None | No physical table |
| Reference_Occupation_PermanentResident | 2 | 0% | ❌ None | 0% mapped; incomplete model |
| Reference_Residence_Type | 8 | 100% | ❌ None | No physical table |
| Reference_Segment | 9 | 100% | ✅ `reference_segment` (empty) | Wrong SA; empty table; auto-optimize missing |
| Reference_Customer_Business_Classification | 8 | 100% | ❌ None | No physical table |
| Reference_Relationship_Manager | 11 | 20% | ❌ None | 80% unmapped; should be in org SA |
| Reference_Credit_Grade | 8 | 100% | ❌ None | No physical table |

---

## 5. Denormalization Register

| Attribute | Host Entity | Originates From | Classification | Justification | Finding |
|---|---|---|---|---|---|
| Primary RM Name | Customer | Reference_Relationship_Manager | **Unnecessary** | RM name should be fetched via FK join to RM reference/organization entity, not denormalized | Remove from Customer; add FK to org SA |
| Primary RM Mobile | Customer | Reference_Relationship_Manager | **Unnecessary** | PII of RM should not be in customer entity | Remove and reference RM entity |
| Primary RM Email | Customer | Reference_Relationship_Manager | **Unnecessary** | PII; repeating group for 3 RM tiers | Remove |
| Secondary/Tertiary RM Code/Name/Mobile/Email (8 attrs) | Customer | Reference_Relationship_Manager | **Unnecessary** | Repeating group of RM tiers (3×4=12 cols) violates 3NF §2.1 | Normalize to `customer_rm_assignment` child table |
| Primary/Secondary Call Back Contact/Number (4 attrs) | Customer | Party/Contact | **Unnecessary** | Repeating group; should be in `Customer_ContactInformation` | Move to contact entity |
| Gross Monthly Income | Customer | Demographics | **Unnecessary** | Also present as `Monthly Income` in Customer_Demographics and `Average Annual Income` in Customer_AdditionalFinancialInformation | Consolidate to Demographics; remove from Customer |
| Monthly Income | Customer_Demographics | Customer | **Unnecessary** | Duplicate of Gross Monthly Income in parent Customer entity | Remove one |
| Annual income | Customer_Demographics | Customer | **Unnecessary** | Approximate duplicate | Consolidate |
| Industry Type, Sector, Sub Sector | Customer | Reference_SIC_Code | **Acceptable** | Commonly denormalized for query performance; needs formal SIC code FK | Document FK; justify or normalize |
| Nationality (from Nationality Code) | Customer_Relationships | Reference_Country | **Unnecessary** | Derived from code lookup — violates SLV-006 (no derived values) | Remove description; keep code only |
| Country of Residence | Customer_Relationships | Reference_Country | **Unnecessary** | Derived from `Country of Residence Code` | Remove |
| Document Code Description | Customer_Documents | Reference document code | **Unnecessary** | Derived from Document Code — SLV-006 violation | Remove description column |
| Document Type Description | Customer_Documents | Reference document type | **Unnecessary** | Derived — SLV-006 violation | Remove |
| TIN Number | Customer_Demographics + Customer_Preferences + Customer_Relationships | — | **Unnecessary (triplication)** | Same PII field in 3 entities without consolidation | Consolidate to one canonical location |
| CRS Undocumented Flag | Customer_Documents + Customer_Relationships | — | **Unnecessary (duplication)** | Same flag in 2 entities | Consolidate to Documents |
| US Relation | Customer_Demographics + Customer_Preferences + Customer_Relationships | — | **Unnecessary (triplication)** | FATCA field triplicated | Move to risk SA; single location |
| Business Center Name | Customer_ContactInformation | silver.org | **Unnecessary** | Should be FK to `silver.org` branch/business center entity | Replace with `business_center_bk` FK |
| Enable E-Banking / SMS Banking / WAP Banking | Customer | Customer_Preferences | **Unnecessary** | Channel preferences already in Customer_Preferences entity | Remove from Customer; resolve with Preferences |

---

## 6. Guideline Compliance Summary

| Rule | Description | Status | Entities Affected | Finding |
|---|---|---|---|---|
| **SLV-001** | `<entity>_sk` (MD5) on every entity | ❌ FAIL | All 31 | No surrogate key in any entity (ID-001) |
| **SLV-002** | `<entity>_bk` on every entity | ❌ FAIL | All 31 | No business key column named `_bk` (ID-002) |
| **SLV-003** | SCD-2 default; SCD-1 needs steward sign-off | ❌ FAIL | All 11 core | SCD columns misnamed/mistyped; appendOnly prevents SCD-2 (SCD-001) |
| **SLV-004** | No source-system codes; use `silver.reference.code_mapping` | ❌ FAIL | Customer, Demographics, Documents, Relationships | Raw Finacle codes mapped straight-through (ETL-002) |
| **SLV-005** | DQ-gated writes; quarantine table | ⚠️ PARTIAL | All entities | No `data_quality_status` or `dq_issues` columns in LM; quarantine process undocumented |
| **SLV-006** | 3NF; no derived values/aggregations | ❌ FAIL | Customer, Documents, Relationships | `Customer Full Name` derived by concatenation; code descriptions stored alongside codes |
| **SLV-007** | No pre-computed metrics | ❌ FAIL | Psychographics, Groupings, Relationships | Percentage metrics: `Total Liability Pcnt.`, `Total Equity Pcnt.`, `Shareholding Pcnt.`, turnover percentages |
| **SLV-008** | All 9 metadata columns present | ❌ FAIL | All 31 | Missing `run_id`, `source_ingestion_date`; others misnamed/mistyped |
| **SLV-009** | All timestamps UTC | ⚠️ UNVERIFIABLE | All | All date/timestamp columns typed STRING or CHAR(18); UTC enforcement cannot be verified |
| **SLV-010** | Monetary amounts in smallest unit (fils for AED) | ❌ FAIL | Customer, Demographics, AdditionalFinancial, Psychographics | All monetary amounts typed STRING; no DECIMAL(18,4) or BIGINT (fils) |
| **NAM-001** | snake_case, lowercase, no reserved words | ❌ FAIL | All 31 | All LM attribute names use Title Case with spaces; camelCase on metadata cols |
| **NAM-002** | Catalog: `silver.<subject_area>.<entity>` | ❌ FAIL | All 10 tables | Physical catalog is `silver_dev_v2.customer` — wrong catalog name (`silver_dev_v2` vs `silver`) |
| **NAM-003** | Table naming: `<entity>` for Silver | ⚠️ PARTIAL | `customer_contactinformation` | Should be `customer_contact_information` (missing underscore) |
| **NAM-005** | No SQL reserved words | ❌ FAIL | Customer_Preferences | Attribute named `Type` — SQL reserved word |
| **DQ-001** | Deduplication on business keys | ⚠️ UNVERIFIABLE | All | No `record_hash` in LM; deduplication pipeline undocumented |
| **DQ-002** | Monetary columns DECIMAL(18,4) | ❌ FAIL | Multiple | All monetary columns are STRING |
| **DQ-003** | PII masking (EID, Passport, Phone, Card) | ❌ FAIL | Documents, ContactInformation, Relationships | Passport Number, EID Number, Mobile Number, TIN not masked |
| **DQ-004** | Error rate threshold 5% → FAIL pipeline | ⚠️ UNVERIFIABLE | All | No DQ status columns in LM |
| **Guideline §2.1** | 3NF — no repeating groups/numbered columns | ❌ FAIL | Customer_ContactInformation | `mail_address1–4`, `registered_address1–4` — 8 numbered columns |
| **Guideline §2.7** | Mandatory Silver technical audit columns | ❌ FAIL | All 11 core | `run_id`, `source_ingestion_date` absent; others misnamed |
| **Guideline §2.11** | Erwin model before DDL | ❌ FAIL | `credit_limit_liability` | Physical table with no logical entity |
| **Guideline §2.12** | Data catalog registration | ⚠️ PARTIAL | 9 of 10 tables | Only `customer` has a description registered; others have `null` descriptions |

---

## 7. Remediation Plan

### 7.1 P0 — Immediate Actions (Current Sprint)

| # | Finding | Action | Owner |
|---|---|---|---|
| P0-01 | SA-001 | Convene data steward and architecture board to decide: (a) merge `silver.customer` into `silver.party`, or (b) formally register `customer` as a new SA in `sa/subject-areas.md` with schema `silver.customer`. Without this decision, all downstream work is at risk. | Data Steward + Architecture |
| P0-02 | ID-001/002 | Add `customer_sk STRING NOT NULL` (= `MD5(UPPER(TRIM(customer_bk)))`) and `customer_bk STRING NOT NULL` to the `Customer` entity and propagate `customer_bk` as FK to all child entities. Repeat for all entities (`<entity>_sk`, `<entity>_bk`). | Data Modeler |
| P0-03 | SCD-001 | Rename `Business Start Date` → `effective_from TIMESTAMP NOT NULL`. Rename `Business End Date` → `effective_to TIMESTAMP`. Replace `IsLogicallyDeleted STRING` → `is_current BOOLEAN NOT NULL`. Add `run_id STRING` and `source_ingestion_date TIMESTAMP`. Apply to all 11 core entities. | Data Modeler + DBA |
| P0-04 | SCD-001 | Remove `appendOnly` Delta table feature from all 9 affected tables to permit SCD-2 MERGE operations. | Data Engineer |
| P0-05 | ETL-002 | Intercept all "straight through" mappings for code fields (Customer Type, Customer Status, Industry Type, Document Type, Gender, Marital Status, etc.) and route through `silver.reference.code_mapping` lookup. | Pipeline Engineer |
| P0-06 | ETL-001 | Develop source-to-target mapping for `Customer_Individual`, `Customer_Business`, and complete the 75 unmapped attributes in the `Customer` entity (especially: VIP Indicator, Customer Status, Customer Since, Non Resident Indicator, Credit Grading Code, Dormant flag). | Mapping Analyst |
| P0-07 | DQ-003 | Immediately implement SHA-256 masking for: Passport Number (`Customer_Documents`), EID Number (`Customer_Documents`), Mobile Number (`Customer_ContactInformation`), TIN Number (all 3 entities). This is a regulatory/PDPL compliance requirement. | Data Security + Pipeline Engineer |
| P0-08 | ETL-001 | Fix mapping document quality issues: rename column `Enitity_Name` → `Entity_Name`; fix entity name `Customer_Relatioships` → `Customer_Relationships`; fix `Souce_System.Column_Data_Type` → `Source_System.Column_Data_Type`. | Mapping Analyst |
| P0-09 | Metadata | Add missing DQ metadata columns to all entity LM definitions: `record_hash STRING`, `batch_id STRING`, `data_quality_status STRING`, `dq_issues STRING`. | Data Modeler |

### 7.2 P1 — Next Sprint

| # | Finding | Action | Owner |
|---|---|---|---|
| P1-01 | SA-002 | Migrate 20 reference entities from `silver.customer` to `silver.reference` schema. Update physical tables and pipeline references. | Data Engineer |
| P1-02 | SA-003 | Investigate `credit_limit_liability` table: either create logical entity in LM or migrate to `silver.account`/`silver.lending`. | Data Steward |
| P1-03 | RI-001 | Define FK relationships in logical model: `customer_bk` FK from each child entity to `Customer`. Add referential integrity checks to Silver pipeline. | Data Modeler |
| P1-04 | ETL-003 | Document conflict resolution logic for multi-source attributes (Finacle + RLS Loan). Implement companion tables per guideline §2.10 or document precedence rules. | Pipeline Architect |
| P1-05 | 3NF | Normalize `Customer_ContactInformation`: replace `mail_address1–4` and `registered_address1–4` with a child `customer_address` table having columns: `address_sk`, `customer_bk` FK, `address_type_code`, `address_line_1`, `address_line_2`, `city`, `emirate_code`, `postal_code`, `country_code CHAR(3)`, `is_primary BOOLEAN`. | Data Modeler |
| P1-06 | Types | Fix all data types in logical model: dates → DATE; timestamps → TIMESTAMP; amounts → BIGINT (fils) or DECIMAL(18,4); booleans → BOOLEAN; codes → STRING (but mark for canonicalization). | Data Modeler |
| P1-07 | INF-001 | Add partition column `_ingested_date DATE` to all 10 physical tables and rebuild. For SCD-2 tables, consider `effective_from::DATE` as secondary partition. | Data Engineer |
| P1-08 | INF-002 | Enable `delta.autoOptimize.autoCompact=true` and `delta.autoOptimize.optimizeWrite=true` on the 7 tables missing it. | Data Engineer |
| P1-09 | SLV-007 | Remove percentage metrics from Silver entities: `Total Liability Pcnt.` and `Total Equity Pcnt.` from `Customer_Relationships`; `Shareholding Pcnt.` from `Customer_Groupings`; `Expected Monthly Cash/Non-Cash Credit Turnover Percentage` from `Customer_Psychographics`. Store raw additive components instead. | Data Modeler |
| P1-10 | Denorm | Create `customer_rm_assignment` child table to replace 12 repeating RM-tier columns in `Customer` entity. | Data Modeler |
| P1-11 | NAM-001 | Rename all logical model attributes from Title Case with spaces to `snake_case`: e.g., `Customer number (CIF)` → `customer_bk`, `CIF Creation Date` → `cif_creation_date`, `IsLogicallyDeleted` → `is_current`. Update mapping document to match. | Data Modeler |
| P1-12 | SA scope | Move `Customer_Psychographics` (KYC/AML data) to `silver.risk` SA. Coordinate with Risk SA data steward. | Architecture + Data Steward |
| P1-13 | Catalog | Register table descriptions in Unity Catalog for all 9 tables currently showing `null` description (`reference_country`, `reference_customer_type`, `customer_documents`, `customer_preferences`, `customer_contactinformation`, `reference_segment`, `credit_limit_liability`, `customer_demographics`, `customer_relationships`). | Data Steward |
| P1-14 | INF-004 | Materialize empty tables: load initial data for `customer_contactinformation`, `customer_preferences`, `customer_relationships`, `reference_segment`. | Pipeline Engineer |

### 7.3 P2 — Backlog

| # | Finding | Action |
|---|---|---|
| P2-01 | SCD-002 | Document SCD strategy per entity (SCD-1 vs SCD-2). Obtain data steward approval for any SCD-1 exceptions (reference tables). |
| P2-02 | Denorm-TIN | Consolidate `TIN Number` to single canonical location (recommend: `Customer_Documents` or `Customer_Demographics`). Remove from `Customer_Preferences` and `Customer_Relationships`. |
| P2-03 | NAM-005 | Rename `Type` attribute in `Customer_Preferences` (SQL reserved word per NAM-005). Propose `fatca_type_code`. |
| P2-04 | SLV-006 | Remove `Customer Full Name` as a concatenated/derived column. Store `first_name`, `middle_name`, `last_name` separately; compute full name in Gold or Semantic layer. |
| P2-05 | Catalog | Register all Silver tables in Informatica GDGC with data classification, data steward, and lineage links per guideline §2.12. |
| P2-06 | SubTypes | Populate `Customer_Individual` and `Customer_Business` sub-type entities with their respective discriminating attributes (move individual-specific attributes from `Customer` to `Customer_Individual`; corporate-specific to `Customer_Business`). |
| P2-07 | ETL-002 | Complete canonicalization of all code fields across all entities through `silver.reference.code_mapping`. Document mappings for each source system code domain. |
| P2-08 | Access | Audit Unity Catalog access controls; ensure no analyst/BI accounts have direct read on `silver_dev_v2.customer.*` for PII-bearing tables. |
| P2-09 | INF | Set up weekly VACUUM schedule on all Delta tables to prevent unbounded file version growth. |

### 7.4 Suggested Schedule

| Sprint | Focus | Key Deliverables |
|---|---|---|
| Sprint 1 (Week 1–2) | Architecture decision + Identity | P0-01: SA taxonomy decision; P0-02: SK/BK added to all entities; P0-07: PII masking |
| Sprint 2 (Week 3–4) | SCD + Metadata | P0-03/04: SCD columns fixed, appendOnly removed; P0-09: DQ metadata columns added |
| Sprint 3 (Week 5–6) | Mapping completion + canonicalization | P0-05/06/08: mapping gap closed to 80%+; code canonicalization via reference |
| Sprint 4 (Week 7–8) | Infrastructure + normalization | P1-01 through P1-06: reference SA migration, FK definition, 3NF normalization, data types |
| Sprint 5 (Week 9–10) | Pipeline + data loading | P1-07 through P1-14: partitioning, auto-optimize, empty table loading |
| Sprint 6 (Backlog) | Governance | P2-01 through P2-09: catalog registration, SCD documentation, sub-type entities |

---

## 8. Appendix

### Appendix A: Mapping Summary

#### A.1 Core Entity Mapping Completeness

| Entity | LM Attributes | Mapping Rows | Mapped (Y) | Unmapped (N) | Not In Map | Coverage |
|---|---|---|---|---|---|---|
| Customer | 145 | 135 | 55 | 80 | 10 | **41%** |
| Customer_AdditionalFinancialInformation | 25 | 14 | 3 | 11 | 11 | **21%** |
| Customer_ContactInformation | 30 | 22 | 15 | 7 | 8 | **68%** |
| Customer_Demographics | 36 | 27 | 8 | 19 | 9 | **30%** |
| Customer_Documents | 28 | 17 | 12 | 5 | 11 | **71%** |
| Customer_Groupings | 16 | 5 | 4 | 1 | 11 | **80%** |
| Customer_Preferences | 51 | 40 | 3 | 37 | 11 | **8%** |
| Customer_Psychographics | 26 | 22 | 14 | 8 | 4 | **64%** |
| Customer_Relationships | 40 | 32 | 9 | 23 | 8 | **28%** |
| Customer_Individual | 5 | 0 | 0 | 0 | 5 | **0%** |
| Customer_Business | 5 | 0 | 0 | 0 | 5 | **0%** |
| **Core Total** | **407** | **314** | **123** | **191** | **93** | **~39%** |

#### A.2 Source Systems

| Source System | Schema | Key Tables | Entities Sourced |
|---|---|---|---|
| Finacle (CBS) | CRMUSER | ACCOUNTS, SALES | Customer, Demographics, Documents, ContactInformation, Groupings, Relationships |
| Finacle (CBS) | CRMUSER | CFMAST, CMAST | ContactInformation (address preferences) |
| Finacle (CBS) | CRMUSER | BRANCH, BRANCHDETAILS | Customer (Domicile Branch transformation) |
| RLS (Loan System) | — | — | Customer (multiple fields: CIF Creation Date, Customer Full Name, Last Name, Domicile Branch, Business Line, Relationship Start Date) |

#### A.3 Mapping Document Quality Issues

| Issue | Location | Impact |
|---|---|---|
| Column header typo: `Enitity_Name` | Customer Silver Data Mapping, col A | Entity filtering errors |
| Entity name typo: `Customer_Relatioships` | Customer Silver Data Mapping, rows 270-301 | FK join failures if used programmatically |
| Column header typo: `Souce_System.Column_Data_Type` | Customer Silver Data Mapping, col L | Schema validation failures |
| Mapping sheet `Customer Silver Refinment Map` has only 1 data row | Customer Silver Refinment Map | Indicates work-in-progress state |

---

### Appendix B: Guideline Citations

| Rule Code | Source Document | Exact Rule Text |
|---|---|---|
| SLV-001 | Assessment Framework | "`<entity>_sk` (MD5) on every entity" |
| SLV-002 | Assessment Framework | "`<entity>_bk` on every entity" |
| SLV-003 | Assessment Framework | "SCD-2 default; SCD-1 needs steward sign-off" |
| SLV-004 | Modeling DOs & DON'Ts §2.5 | "DO: Resolve all source-system codes to canonical platform values using a central code-mapping reference (e.g., `silver.reference.code_mapping`). Store the canonical code, not the raw source code, in entity columns." |
| SLV-006 | Modeling DOs & DON'Ts §2.6 | "DO: Store only atomic, un-aggregated facts in Silver. DON'T: Pre-compute totals, averages, counts, ratios, or any derived metrics in Silver tables." |
| SLV-007 | Modeling DOs & DON'Ts §2.6 | "Improper — pre-aggregated metric in Silver: `total_transactions_30d INT — aggregation; should not be in Silver`" |
| SLV-008 | Modeling DOs & DON'Ts §2.7 | "Include the following Silver technical audit columns on every Silver entity table: `source_system_code`, `source_system_id`, `create_date`, `update_date`, `delete_date`, `is_active_flag`, `effective_from`, `effective_to`, `is_current`, `run_id`, `source_ingestion_date`" |
| SLV-009 | Modeling DOs & DON'Ts §2.9 | "Store all timestamps in UTC. Convert source-local timestamps at ingestion." |
| SLV-010 | DQ Controls §4 | "All monetary columns must use `DECIMAL(18,4)`" |
| NAM-001 | Naming Conventions | "Use snake_case for all identifiers (catalogs, schemas, tables, columns, views). Use lowercase only — no mixed case." |
| NAM-005 | Naming Conventions | "Never use SQL reserved words as identifiers (e.g., `date`, `value`, `order`)." |
| 3NF | Modeling DOs & DON'Ts §2.1 | "DON'T: Add numbered or suffixed columns to represent multiple instances of the same concept (e.g., `address1`, `address2`, `address3`, `address4`; `phone1`, `phone2`; `email_1`, `email_2`)." |
| PII Masking | DQ Controls §2 | "PII Masking: EID, Card Numbers, Phone Numbers, and Passport Numbers masked via SHA-256 unless stored in a restricted container" |
| PDPL | DQ Controls §10 | "Right to Be Forgotten: Automated pipelines must support erasure of customer data within 30 days" |
| Surrogate Key | Modeling DOs & DON'Ts §2.4 | "Derive surrogate keys deterministically: `MD5(UPPER(TRIM(business_key)))` for single-component keys." |
| SCD-2 | Modeling DOs & DON'Ts §2.3 | "Track all changes to tracked attributes with SCD Type 2: add a new row with updated values, close the prior row (`effective_to` = change timestamp, `is_current` = FALSE). Every Silver entity table must have `effective_from`, `effective_to`, `is_current`." |
| Catalog | DQ Controls §5 | Mandatory audit columns: `record_source`, `record_origin`, `record_hash`, `is_current`, `valid_from_at`, `valid_to_at`, `ingested_at`, `processed_at`, `batch_id`, `data_quality_status`, `dq_issues` |
| SA Isolation | Modeling DOs & DON'Ts §2.2 | "Keep each Silver subject area self-contained. Pipelines for one subject area read only from Bronze and from the same subject area's own Silver tables." |
| Source Fidelity | Modeling DOs & DON'Ts §2.11 | "Tables that exist in the platform but have no corresponding entity in the Erwin model are a governance gap." |

---

### Appendix C: Industry References

| Standard | Version | Application in This Assessment |
|---|---|---|
| **BIAN** (Banking Industry Architecture Network) | v11 | Party Data Management service domain used to evaluate Customer vs Party entity separation; Customer Management domain used to evaluate role-based attributes |
| **IFW** (IBM/Teradata Industry Framework for Banking) | Current | Party vs Customer role distinction; PartyRole pattern applied to evaluate sub-type modeling of Customer_Individual / Customer_Business |
| **BCBS 239** | 2013 Principles | Data lineage completeness requirement: principle 2 (Data Architecture) applied to SA scoping; principle 6 (Completeness) applied to metadata columns |
| **IFRS 9** | 2014/2018 | No direct IFRS 9 attributes found in customer SA; ECL staging attributes should be in `silver.lending` |
| **UAE PDPL** | Federal Decree-Law No. 45 of 2021 | Article 5 processing prohibition for opted-out customers; right-to-be-forgotten (30 days) cited in DQ Controls §10 |
| **FATCA** | IRS Rev. Rul. 2014 | Multiple FATCA attributes (TIN Number, US Relation, GIIN, Financial Entity) found in Customer_Preferences / Customer_Relationships / Customer_Demographics — recommend consolidation to `silver.risk` |
| **CRS** (Common Reporting Standard) | OECD 2014 | CRS Undocumented Flag, CRS Undocumented Flag Reason found in Customer_Documents and Customer_Relationships — consolidate to Customer_Documents |
| **Databricks Delta Lake** | Current | appendOnly table feature, ChangeDataFeed, columnMapping, rowTracking evaluated per SCD-2 compatibility |
| **Kimball** (Ralph Kimball) | *The Data Warehouse Toolkit* 3rd ed. | Conformed dimensions principle applied to reference entity scoping; surrogate key pattern evaluated |
| **3NF** (Codd) | Original definition | First/Third Normal Form violations: repeating groups (address1–4), transitive dependencies (code + description in same entity) |
