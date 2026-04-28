# Gap Assessment Report — Customer Subject Area (Silver Layer)

**Assessment Date**: 2026-04-28  
**Subject Area**: `Customer`  
**Physical Catalog / Schema**: `silver_dev_v2.customer` *(non-canonical — see SA-001)*  
**Canonical Schema per SA Taxonomy**: `silver.party` *(Customer is a Party role)*  
**Guideline Versions**: Modeling DOs & DON'Ts v1.0 Draft · Naming Conventions v1.0 Draft · DQ Controls v1.0 Draft  
**Total Logical Entities**: 31 (11 core + 20 reference)  
**Total Logical Attributes**: 584  
**Physical Tables Found**: 10  

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)  
2. [Task 1 — Subject Area Architecture Review](#2-task-1--subject-area-architecture-review)  
3. [Task 2 — Missing Critical Entities Check](#3-task-2--missing-critical-entities-check)  
4. [Task 3 — Per-Entity Assessment](#4-task-3--per-entity-assessment)  
5. [Task 4 — Denormalization Analysis](#5-task-4--denormalization-analysis)  
6. [Guideline Compliance Summary](#6-guideline-compliance-summary)  
7. [Remediation Plan](#7-remediation-plan)  
8. [Appendix](#8-appendix)  

---

## 1. Executive Summary

### 1.1 Management Dashboard

| Metric | Value |
|--------|-------|
| Total Logical Entities Assessed | 31 (11 core + 20 reference) |
| Physical Tables Deployed | 10 |
| Logical Entities Without a Physical Table | 23 |
| Orphan Physical Tables (no logical entity) | 1 (`credit_limit_liability`) |
| Total Logical Model Attributes | 584 |
| Attributes Typed Correctly (non-STRING where needed) | ~30 / 584 (~5%) |
| Attributes Mapped to a Source Column | ~152 / 584 (~26%) |
| **P0 Findings** | **18** |
| **P1 Findings** | **15** |
| **P2 Findings** | **9** |
| **Total Findings** | **42** |
| Overall Guideline Compliance Score | **~18%** |

### 1.2 Top 5 Priority Actions

1. **[P0 — Architecture]** The `customer` subject area does not exist in the canonical SA taxonomy. `sa/subject-areas.md` Structural Decisions states: *"Customer is a Party role, not a separate SA — Avoids duplication of identity data; KYC and segmentation live in `silver.party`."* The team must immediately convene an architecture decision: either merge `silver.customer` into `silver.party` or formally register and justify it as a new SA.
2. **[P0 — Identity]** Not one of the 31 logical entities carries a `<entity>_sk` (MD5 deterministic surrogate key) or a properly named `<entity>_bk` (business key). This violates Guideline §2.4 universally and makes all downstream SCD-2 and Gold FK joins impossible.
3. **[P0 — Data Types]** 554 of 584 attributes (95%) are typed `STRING`; the remaining 30 use `CHAR(18)`. No date columns use `DATE` or `TIMESTAMP`, no monetary columns use `DECIMAL(18,4)` or `BIGINT`, and no boolean indicators use `BOOLEAN`. This is a systemic breach of DQ Controls §4 and Guideline §2.7.
4. **[P0 — Metadata]** Mandatory Silver audit columns are absent, mis-named, or mis-typed across all entities. `run_id` and `source_ingestion_date` are entirely absent from the logical model. SCD-2 columns appear as `Business Start Date` / `Business End Date` typed `CHAR(18)` — wrong name and wrong type. `is_current` is modelled as `IsLogicallyDeleted STRING` — inverted semantics and camelCase naming violation.
5. **[P0 — PII]** `Passport Number` and `EID Number` in `Customer_Documents`, `Mobile Number` in `Customer_ContactInformation`, and `TIN Number` across three entities are all unmapped and carry no SHA-256 masking directive. DQ Controls §2 mandates masking for all these fields before Silver promotion.

---

## 2. Task 1 — Subject Area Architecture Review

### 2.1 Domain Scoping & SA Taxonomy Alignment

**Finding SA-001 | P0 | High | Confidence: 1.00**  
**`customer` SA is not registered in the canonical SA taxonomy**

`sa/subject-areas.md` contains 19 subject areas. None is named `customer`. The Structural Decisions table explicitly states:

> *"Customer is a Party role, not a separate SA — Avoids duplication of identity data; KYC and segmentation live in `silver.party`."*

The physical catalog `silver_dev_v2.customer` is running against this explicit architectural decision, creating:
- **Identity duplication risk** between `silver.party` (canonical) and `silver.customer` (current).
- **Consumer confusion** downstream: Gold/Semantic layers have two competing sources of customer truth.
- **Schema name non-conformance**: NAM-002 requires `silver.<subject_area>.<entity>`; `silver_dev_v2` is a development alias, not the production catalog.

**BIAN v11 alignment**: The applicable service domains are *Party Data Management* and *Customer Management*. BIAN does not model "Customer" as a standalone top-level domain independent of "Party". The entities `Customer_Individual` and `Customer_Business` directly correspond to BIAN Party sub-types (Individual Party, Organisation Party). All sub-entities (contact, documents, demographics, relationships) are BIAN *Party Reference Data* components.

**IFW alignment**: IFW separates *Party* (universal legal entity: name, identity, documents) from *Customer* (a role a Party plays at the bank: segment, RM assignment, VIP status). The current `Customer` entity conflates both — 145 attributes mixing party identity with customer-role attributes.

**Remediation**: Architecture board to decide within current sprint: (a) migrate into `silver.party` or (b) formally register `customer` as SA #20 in `sa/subject-areas.md` with explicit justification. All downstream work is blocked until this decision is made.

---

**Finding SA-002 | P1 | High | Confidence: 1.00**  
**20 reference entities are scoped to `customer` SA — belong in `silver.reference`**

The canonical SA for lookup/reference data is `silver.reference` (SA #1, Priority 1 Critical). The 20 `Reference_*` entities below are incorrectly placed in `silver.customer`:

`Reference_SIC_Code`, `Reference_SIC_Group`, `Reference_SIC_EconomicMeasure`, `Reference_Business_SIC_History`, `Reference_Customer_Status`, `Reference_Customer_Type`, `Reference_Country`, `Reference_Ethnicity`, `Reference_Gender`, `Reference_Organization_Type`, `Reference_ID_Type`, `Reference_Education`, `Reference_Own_Home`, `Reference_Marital_Status`, `Reference_Occupation_PermanentResident`, `Reference_Residence_Type`, `Reference_Segment`, `Reference_Customer_Business_Classification`, `Reference_Relationship_Manager`, `Reference_Credit_Grade`

Of these, only `reference_country`, `reference_customer_type`, and `reference_segment` have physical tables (all in wrong schema). The remaining 17 are unmaterialized.

**Note on `Reference_Relationship_Manager`**: This entity (RM code, employee ID, department ID) belongs in `silver.org` (Organisation SA #18), not reference data and certainly not customer SA.

**Remediation**: Migrate 3 physical tables to `silver.reference`. Remove all 20 from customer SA logical model scope.

---

**Finding SA-003 | P1 | Medium | Confidence: 0.90**  
**`credit_limit_liability` physical table has no logical entity**

The physical table `silver_dev_v2.customer.credit_limit_liability` exists in Unity Catalog with no corresponding logical entity, no mapping rows, and no catalog description (`null`). Guideline §2.11: *"Tables that exist in the platform but have no corresponding entity in the Erwin model are a governance gap."* The entity semantically belongs in `silver.account` or `silver.lending`, not `silver.customer`.

---

### 2.2 Entity Set Completeness vs. BIAN / IFW Decomposition

The subject area attempts 11 core entities. Evaluated against the BIAN Party Data Management domain and IFW Party/Customer decomposition:

| IFW / BIAN Component | Present? | Notes |
|---|---|---|
| Party (identity root) | ✅ `Customer` (mis-named) | Conflates Party + CustomerRole |
| Party Sub-type: Individual | ✅ `Customer_Individual` | Empty shell — no individual attributes |
| Party Sub-type: Organisation | ✅ `Customer_Business` | Empty shell — no corporate attributes |
| Party Contact | ✅ `Customer_ContactInformation` | 3NF violation: numbered address cols |
| Party Document | ✅ `Customer_Documents` | PII masking absent |
| Party Demographics | ✅ `Customer_Demographics` | Income duplicated in parent |
| Customer Role | ❌ absent | Mixed into `Customer` entity |
| Party Relationship | ✅ `Customer_Relationships` | Relationship type unmapped |
| Customer Group Membership | ✅ `Customer_Groupings` | No physical table |
| Customer Preference / Consent | ✅ `Customer_Preferences` | FATCA mixed in; 6% mapped |
| KYC / AML Profile | ✅ `Customer_Psychographics` | Misnamed; belongs in `silver.risk` |
| Financial Profile | ✅ `Customer_AdditionalFinancialInformation` | Risk attributes mixed in; no physical table |
| **Party Address** (normalized) | ❌ absent | 3NF split of contact not done |
| **Party Identifier** (multi-doc) | ❌ absent | Multi-document type collapsed into Documents |
| **Customer RM Assignment** | ❌ absent | Denormalized as repeating group in Customer |
| **Consent Record** (PDPL) | ❌ absent | Scattered across Preferences + Customer |

---

### 2.3 SCD Strategy

**Finding SCD-001 | P0 | High | Confidence: 1.00**  
**SCD-2 columns are mis-named, mis-typed, and semantically inverted across all entities**

Guideline §2.3: *"Every Silver entity table must have `effective_from`, `effective_to`, `is_current`."*

| Required Column | Actual in Logical Model | Type in LM | Issue |
|---|---|---|---|
| `effective_from` | `Business Start Date` | `CHAR(18)` | Wrong name; wrong type (must be TIMESTAMP NOT NULL) |
| `effective_to` | `Business End Date` | `CHAR(18)` | Wrong name; wrong type (must be TIMESTAMP nullable) |
| `is_current` | `IsLogicallyDeleted` | `STRING` | Wrong name (camelCase); inverted semantics; wrong type (must be BOOLEAN NOT NULL) |
| `run_id` | *(absent)* | — | Entirely missing from all 31 entities |
| `source_ingestion_date` | *(absent)* | — | Entirely missing from all 31 entities |

Additionally, 9 of 10 physical tables carry the `appendOnly` Delta feature, which physically prevents the `DELETE`/`UPDATE` operations that SCD-2 MERGE requires. SCD-2 is currently impossible on the physical layer.

**Finding SCD-002 | P1 | Medium | Confidence: 0.85**  
**No documented SCD type assignment for any entity**

No entity in the logical model or mapping document specifies SCD-1 vs SCD-2. Reference entities likely warrant SCD-1 (overwrite); core customer entities require SCD-2. Guideline §2.3 requires explicit data-steward approval for any SCD-1 exception.

---

### 2.4 Cross-Entity Referential Integrity

**Finding RI-001 | P0 | High | Confidence: 1.00**  
**No foreign keys defined anywhere in the logical model**

All child entities carry `Customer number (CIF)` and `Customer Type` as composite PK references, but:
- Neither column is named `customer_sk` or `customer_bk` — no FK naming convention applied.
- No referential integrity is formally documented or enforced.
- DQ Controls §3.1: *"Referential Integrity: All foreign keys must reference valid dimension records."*

**Finding RI-002 | P1 | High | Confidence: 0.90**  
**`Customer_Relationships` self-join has no FK and discriminator is unmapped**

`CIF ID` and `Related CIF ID` imply a self-join to `Customer`. No FK is defined. `Relationship Type` — the critical discriminator for classifying UBO, guarantor, household member, authorized signatory — has mapping status `N` (unmapped).

---

### 2.5 Mapping & Lineage

**Finding ETL-001 | P0 | High | Confidence: 1.00**  
**Overall mapping completeness is ~26% — most of the SA is unmapped**

| Entity | LM Attrs | Mapped (Y) | Unmapped (N) | Coverage |
|---|---|---|---|---|
| Customer | 145 | 55 | 90 | 38% |
| Customer_AdditionalFinancialInformation | 25 | 3 | 22 | 12% |
| Customer_ContactInformation | 30 | 15 | 15 | 50% |
| Customer_Demographics | 36 | 8 | 28 | 22% |
| Customer_Documents | 28 | 12 | 16 | 43% |
| Customer_Groupings | 16 | 4 | 12 | 25% |
| Customer_Preferences | 51 | 3 | 48 | 6% |
| Customer_Psychographics | 26 | 14 | 12 | 54% |
| Customer_Relationships | 40 | 9 | 31 | 23% |
| Customer_Individual | 5 | 0 | 5 | 0% |
| Customer_Business | 5 | 0 | 5 | 0% |
| **Core Total** | **407** | **123** | **284** | **~30%** |

**Finding ETL-002 | P0 | High | Confidence: 0.95**  
**Code fields mapped straight-through without canonicalization**

Guideline §2.5: *"DO: Resolve all source-system codes to canonical platform values using `silver.reference.code_mapping`. DON'T: Store source-system-specific codes directly in Silver."*

The mapping document shows `Customer Type` (from `CRMUSER.ACCOUNTS.CUST_TYPE`), `Industry Type` (SIC codes), and several status fields as "straight through" with no transformation. Raw Finacle CBS codes will land in Silver uncanonicalised.

**Finding ETL-003 | P1 | Medium | Confidence: 0.85**  
**Multi-source conflict resolution undocumented (Finacle + RLS Loan)**

The mapping document's `Fields Classification` sheet lists multiple attributes with `Source of change: Multiple` from both Finacle and RLS Loan System (e.g., `CIF Creation Date`, `Customer Full Name`, `Domicile Branch`). No conflict resolution / precedence logic is documented. Guideline §2.10 requires companion tables per source system; the current design plans a single wide entity.

---

### 2.6 Physical Infrastructure

**Finding INF-001 | P1 | Medium | Confidence: 1.00**  
**No partitioning on any of the 10 physical tables**

All 10 tables have `partitionColumns: []`. `customer` (244 MB) and `customer_demographics` (106 MB) are high-volume tables that will degrade without partition pruning. Guideline §1.4: *"Partition all tables by ingestion date."* At Silver, `effective_from::date` or `_ingested_date` are the preferred partition keys.

**Finding INF-002 | P1 | Medium | Confidence: 1.00**  
**Auto-optimize missing on 7 of 10 tables**

Tables missing `delta.autoOptimize.autoCompact` and `delta.autoOptimize.optimizeWrite`: `reference_country`, `reference_customer_type`, `customer_contactinformation`, `reference_segment`, `credit_limit_liability`, `customer_demographics`, `customer_relationships`.

**Finding INF-003 | P1 | Medium | Confidence: 1.00**  
**`appendOnly` Delta feature blocks SCD-2 on 9 tables**

Nine tables carry the `appendOnly` feature (listed in SCD-001 above). This must be removed before any SCD-2 MERGE pipeline can run.

**Finding INF-004 | P2 | Low | Confidence: 1.00**  
**Five tables are empty (0 bytes)**

`customer_preferences`, `customer_contactinformation`, `reference_segment`, `credit_limit_liability`, `customer_relationships` have never been loaded. Pipelines are not implemented for these entities.

**Finding INF-005 | P2 | Low | Confidence: 1.00**  
**Catalog descriptions absent on 9 of 10 tables**

Only `customer` carries a description in Unity Catalog. All other tables have `null` description. Guideline §2.12 requires catalog registration with data classification, steward name, and lineage links.

---

## 3. Task 2 — Missing Critical Entities Check

The following canonical banking entities are absent from or incorrectly collapsed in the Customer logical model.

### 3.1 Normalized Party Address Entity

**Missing Entity: `customer_address`**

- **Why critical**: The current `Customer_ContactInformation` entity uses `mail_address1–4` and `registered_address1–4` — 8 numbered columns that are a repeating group violating 1NF/3NF (Guideline §2.1). This design cannot support: (a) a fifth address without DDL change, (b) address type discrimination (home/work/mailing), (c) emirate-level analytics for UAE regulatory reporting, (d) correct partial updates to individual addresses under PDPL right-to-be-forgotten.
- **Regulatory impact**: CBUAE KYC requires verified address by type. PDPL Article 5 requires targeted erasure of address data without affecting other contact records.
- **Impact on consumers**: Gold dimensions cannot build `dim_customer` with clean address facts; BI cannot filter "customers in Dubai" without LIKE on 4 columns.
- **Priority**: P0 — normalization is a correctness issue, not cosmetic.
- **Remediation**: Create `customer_address` child table with columns: `address_sk`, `customer_bk` (FK), `address_type_code`, `address_line_1`, `address_line_2`, `district`, `city`, `emirate_code`, `postal_code`, `country_code CHAR(3)`, `is_primary BOOLEAN`, plus full SCD-2 audit columns.
- **Effort**: 2–3 days (logical model + DDL + pipeline).

### 3.2 Normalized Party Contact / Channel Entity

**Missing Entity: `customer_contact`**

- **Why critical**: Phone, mobile, and email are single-value columns in `Customer_ContactInformation`. Customers routinely have multiple phone numbers (home/work/mobile) and email addresses. The current design drops all but the "first" contact channel. Guideline §2.1 explicitly shows a `customer_contact` child table as the correct pattern.
- **Regulatory impact**: CBUAE and AML require verified contact channels with verification date. PII masking must be applied to each contact value individually.
- **Priority**: P1.
- **Remediation**: Create `customer_contact` table: `contact_sk`, `customer_bk` (FK), `contact_type_code`, `contact_value` (SHA-256 masked), `is_primary`, `is_verified`, `verified_date`.
- **Effort**: 2 days.

### 3.3 Standalone Consent Record Entity

**Missing Entity: `customer_consent`**

- **Why critical**: Consent flags are scattered across `Customer_Preferences` (Do Not Call, Do Not SMS, Do Not Mail, Do Not Email — 8 indicator flags) and the `Customer` entity (Consent to enquire AECB, Consent Source, Consent Date). There is no single auditable consent record with a consent type, grant/revocation timestamp, channel, and expiry. UAE PDPL Article 5 (cited in DQ Controls §10) requires: *"processing prohibited for opted-out customers"* and *"right to be forgotten within 30 days."* Without a structured consent table, automated erasure pipelines cannot be built.
- **Regulatory impact**: PDPL non-compliance if consent is not trackable per data category.
- **Priority**: P0 — regulatory requirement.
- **Remediation**: Create `customer_consent` table: `consent_sk`, `customer_bk` (FK), `consent_type_code`, `consent_channel_code`, `is_granted`, `granted_at`, `revoked_at`, `expiry_date`, `consent_source_code`, audit columns.
- **Effort**: 2–3 days.

### 3.4 Customer Role Entity (CustomerRole separation)

**Missing Entity: `customer_role`** *(or equivalent discriminator sub-table)*

- **Why critical**: Per IFW and BIAN, a `Party` entity holds identity attributes (name, DoB, nationality) while a `CustomerRole` entity holds the bank-relationship attributes (customer segment, VIP status, domicile branch, RM assignment, relationship start/end dates, business line). The current `Customer` entity conflates 145 attributes from both domains. This means: (a) the grain is ambiguous — is it one row per person or one row per customer relationship? (b) a person who has multiple customer roles (e.g., retail + private banking) cannot be represented correctly.
- **Priority**: P1.
- **Remediation**: Extract role-specific attributes into a `customer_role` entity keyed by `customer_bk` + `role_type_code` with SCD-2 on role-level changes.
- **Effort**: 4–5 days (significant refactoring).

### 3.5 RM Assignment Entity

**Missing Entity: `customer_rm_assignment`**

- **Why critical**: The `Customer` entity contains 12 attributes for three RM tiers (Primary/Secondary/Tertiary Code/Name/Mobile/Email). This is a textbook repeating group violating 3NF (Guideline §2.1). It silently drops a 4th RM if one is ever assigned, and prevents querying "all customers assigned to RM X" without a CASE-based UNION across 3 column sets.
- **Priority**: P1.
- **Remediation**: Create `customer_rm_assignment` child table: `assignment_sk`, `customer_bk` (FK), `rm_tier_code` (PRIMARY/SECONDARY/TERTIARY), `rm_bk` (FK to org SA), `assigned_date`, `end_date`, audit columns.
- **Effort**: 1–2 days.

### 3.6 Party Identifier Entity (multi-document)

**Missing Entity: `customer_identifier`** *(separate from `Customer_Documents`)*

- **Why critical**: `Customer_Documents` mixes the identity document registry (Passport, EID, Trade License) with document workflow metadata (CRS flag, Trade License update status, document verification status). The `Customer` entity also carries `ID number` and `ID type` — duplicating the primary identity document. A separate `customer_identifier` entity is the canonical pattern per IFW Party model for holding multiple government IDs with type/issuer/expiry.
- **Priority**: P1.
- **Remediation**: Create `customer_identifier`: `identifier_sk`, `customer_bk` (FK), `identifier_type_code`, `identifier_value` (SHA-256 masked), `issuing_country_code`, `issue_date`, `expiry_date`, `is_primary`, audit columns.
- **Effort**: 2 days.

---

## 4. Task 3 — Per-Entity Assessment

### 4.1 Entity: `Customer`

#### 4.1.1 Industry Fitness

**Boundary / Grain**: One row per customer CIF (Customer Information File), SCD-2 tracked. The entity intends to be the root entity for all customer master data.

**Scope creep — mixed responsibilities (all are findings):**
| Attribute Group | Count | Correct Location |
|---|---|---|
| Party identity (Full Name, DoB, Nationality, EID, ID number) | ~15 | Party entity or `Customer_Individual`/`Customer_Business` sub-types |
| Customer role (Segment Code, VIP Indicator, RM Code, Business Line) | ~12 | Customer role entity |
| AML / risk (Risk Profile, Credit Grading, Moody's Rating, Credit Bureau Score, Watchlist) | ~8 | `silver.risk` SA |
| RM contact details — repeating group (Primary/Secondary/Tertiary RM Name/Mobile/Email) | 12 | `customer_rm_assignment` child table |
| Channel preferences (Enable E-Banking, SMS Banking, WAP Banking, Phone Banking) | ~6 | `Customer_Preferences` |
| Product eligibility (Available Products, Selected Products) | 2 | `silver.product` or `silver.account` |
| Bank channel (Primary/Secondary Call Back Contact/Number) | 4 | `Customer_ContactInformation` or consent entity |

**3NF violations:**
1. RM tier repeating group: 12 attributes (3 tiers × 4 columns each: Code, Name, Mobile, Email).
2. Call-back contact repeating group: 4 attributes (Primary/Secondary Contact / Number).
3. Consent flags (`Consent held to enquire AECB`, `Consent Source`, `Consent Date`) overlap with `Customer_Preferences`.

#### 4.1.2 Attribute-Level Review

| Attribute | LM Type | Issue |
|---|---|---|
| `Customer number (CIF)` | STRING | Should be `customer_bk STRING NOT NULL` — NAM-001: spaces + parentheses invalid |
| *(absent)* | — | `customer_sk STRING NOT NULL` entirely missing — Guideline §2.4 |
| `Customer Type` | STRING | Raw source code from CRMUSER; must canonicalize via `silver.reference.code_mapping` |
| `CIF Creation Date` | STRING | Should be `DATE`; stored as STRING — DQ §4 |
| `Customer Full Name` | STRING | Concatenated derived value (`CUST_FIRST_NAME \|\| MIDDLE \|\| LAST`); violates Guideline §2.6 (no derived values in Silver) |
| `Birth date` | STRING | Should be `DATE` |
| `Relationship Start Date` | STRING | Should be `DATE` |
| `Customer end date` | STRING | Should be `DATE` |
| `Gross Monthly Income` | STRING | Should be `DECIMAL(18,4)` or `BIGINT` (fils) — DQ §4 monetary standard |
| `Credit Bureau Score` | STRING | Should be `INTEGER` or `DECIMAL`; unmapped |
| `VIP Indicator` | STRING | Should be `BOOLEAN`; rename `is_vip` — NAM-001 |
| `Suspended` | STRING | Should be `BOOLEAN`; rename `is_suspended` |
| `Blacklisted` | STRING | Should be `BOOLEAN`; rename `is_blacklisted` |
| `Minor Indicator` | STRING | Should be `BOOLEAN`; rename `is_minor` |
| `Non Resident Indicator` | STRING | Should be `BOOLEAN`; rename `is_non_resident` |
| `Staff indicator` | STRING | Should be `BOOLEAN`; rename `is_staff` |
| `Dormant` | STRING | Should be `BOOLEAN`; rename `is_dormant` |
| `Primary RM Code` | STRING | Starts 12-attribute repeating group — 3NF violation |
| `Moody's Rating` | STRING | Belongs in `silver.risk` SA |
| `Risk Profile` | STRING | Belongs in `silver.risk` SA |
| `Available Products` | STRING | Belongs in `silver.product`/`silver.account` |
| `Whitelist Flag` | STRING | Should be `BOOLEAN`; `is_whitelisted` |
| `Enable E-Banking` | STRING | Should be `BOOLEAN`; duplicates `Customer_Preferences` |
| `Business Start Date` | CHAR(18) | Must be `effective_from TIMESTAMP NOT NULL` |
| `Business End Date` | CHAR(18) | Must be `effective_to TIMESTAMP` |
| `IsLogicallyDeleted` | STRING | Must be `is_current BOOLEAN NOT NULL` — wrong semantics + camelCase |
| `Is Active Flag` | CHAR(18) | Should be `is_active_flag STRING` |
| *(absent)* | — | `run_id`, `source_ingestion_date`, `record_hash`, `batch_id`, `data_quality_status`, `dq_issues` all missing |
| `SIC Code` | STRING | Mapped `N`; duplicates Reference_SIC_Code; should be FK |

**Mapping**: 55/145 attributes mapped (38%). 10 attributes are not in the mapping document at all (critical metadata columns).

#### 4.1.3 Metadata Completeness

| Column | Present? | Correctly Named? | Correct Type? |
|---|---|---|---|
| `source_system_code` | ✅ (as "Source System Code") | ❌ Title Case | ✅ STRING |
| `source_system_id` | ✅ (as "Source System Id") | ❌ Title Case | ✅ STRING |
| `create_date` | ✅ (as "Create Date") | ❌ Title Case | ❌ STRING→TIMESTAMP |
| `update_date` | ✅ (as "Update Date") | ❌ Title Case | ❌ STRING→TIMESTAMP |
| `delete_date` | ✅ (as "Delete Date") | ❌ Title Case | ❌ STRING→TIMESTAMP |
| `is_active_flag` | ✅ (as "Is Active Flag") | ❌ Title Case | ❌ CHAR(18)→STRING |
| `effective_from` | ✅ (as "Business Start Date") | ❌ Wrong name | ❌ CHAR(18)→TIMESTAMP NOT NULL |
| `effective_to` | ✅ (as "Business End Date") | ❌ Wrong name | ❌ CHAR(18)→TIMESTAMP |
| `is_current` | ✅ (as "IsLogicallyDeleted") | ❌ Wrong name + inverted semantics | ❌ STRING→BOOLEAN NOT NULL |
| `run_id` | ❌ ABSENT | — | — |
| `source_ingestion_date` | ❌ ABSENT | — | — |
| `record_hash` (DQ §5) | ❌ ABSENT | — | — |
| `batch_id` (DQ §5) | ❌ ABSENT | — | — |
| `data_quality_status` (DQ §5) | ❌ ABSENT | — | — |
| `dq_issues` (DQ §5) | ❌ ABSENT | — | — |

**Result: 0 of 15 required metadata columns fully compliant.**

---

### 4.2 Entity: `Customer_AdditionalFinancialInformation`

#### 4.2.1 Industry Fitness

**Boundary**: Financial profile — net worth, income, borrowing history, external banking relationships. Scope issues:
- AML/risk attributes (`Expatriate Risk`, `Overseas Risk`, `IAS Standards`, `Credit Policy Breach`) belong in `silver.risk`.
- External bank data (`Bank Name`, `Branch Name`, `Product Category`, `A/c. ID`) is an embedded reference to an external institution — should be a normalized `external_bank_account` sub-entity or moved to an application/relationship SA.
- `Deposits/Inv/Savings` appears to be a pre-aggregated or freeform string summary — violates Guideline §2.6 (no aggregations in Silver).

**Physical table**: None deployed.

#### 4.2.2 Attribute-Level Review

| Attribute | LM Type | Issue |
|---|---|---|
| `Net worth` | STRING | Should be `DECIMAL(18,4)` or `BIGINT`; rename `net_worth_amount_aed` |
| `Expatriate Risk` | STRING | Belongs in `silver.risk`; should be `BOOLEAN is_expatriate_risk` |
| `Overseas Risk` | STRING | Belongs in `silver.risk` |
| `IAS Standards` | STRING | Ambiguous; belongs in risk SA |
| `Credit Policy Breach` | STRING | Should be `BOOLEAN is_credit_policy_breach`; belongs in `silver.risk` |
| `Borrowing Start Date` | STRING | Should be `DATE` |
| `Average Annual Income` | STRING | Should be `DECIMAL(18,4)` or `BIGINT` |
| `Audited Financials Held` | STRING | Should be `BOOLEAN is_audited_financials_held` |
| `Deposits/Inv/Savings` | STRING | Pre-aggregated or free-text — Guideline §2.6 violation |
| `A/c. ID` | STRING | Non-standard name; should be `account_bk` FK; belongs in account SA |

**Mapping**: 3/25 (12%). Entity not materialized.

#### 4.2.3 Metadata Completeness
Identical deficiencies to §4.1.3. No physical table.

---

### 4.3 Entity: `Customer_ContactInformation`

#### 4.3.1 Industry Fitness

**Boundary**: Customer contact data — addresses, phone, email, preferred contact channel. Critical structural violation: 8 numbered address columns (`Mail address1–4`, `Registered Address1–4`) are a repeating group violating 1NF/3NF.

Guideline §2.1 (verbatim): *"DON'T: Add numbered or suffixed columns to represent multiple instances of the same concept (e.g., `address1`, `address2`, `address3`, `address4`)."*

The guideline explicitly provides a worked DDL example for `customer_address` (normalized, one row per address) and `customer_contact` (one row per contact channel). Neither pattern is implemented.

Additionally, `Phone No. / E-mail` is a single column holding two distinct data types — a structural defect (one column cannot validly contain both a phone number and an email address).

**Physical table**: `customer_contactinformation` (empty, 0 bytes). Physical name violates NAM-003 — should be `customer_contact_information`.

#### 4.3.2 Attribute-Level Review

| Attribute | LM Type | Issue |
|---|---|---|
| `Mail address1` | STRING | **1NF/3NF violation** — numbered repeating group |
| `Mail address2` | STRING | **1NF/3NF violation** |
| `Mail address3` | STRING | **1NF/3NF violation** |
| `Mail address4` | STRING | **1NF/3NF violation** |
| `Registered Address1` | STRING | **1NF/3NF violation** |
| `Registered Address2` | STRING | **1NF/3NF violation** |
| `Registered Address3` | STRING | **1NF/3NF violation** |
| `Registered Address4` | STRING | **1NF/3NF violation** |
| `Phone number` | STRING | PII — SHA-256 masking required per DQ §2 |
| `Mobile Number` | STRING | PII — SHA-256 masking required |
| `Email id` | STRING | PII — masking required; rename `email_address` |
| `Phone No. / E-mail` | STRING | Invalid multi-type column; must be split |
| `Preferred address based on CFMAST` | STRING | Source-system name leaking into Silver — SLV-004 |
| `Preferred address based on CMAST` | STRING | Source-system name leaking — SLV-004 |
| `Preferred Flag` | STRING | Should be `BOOLEAN is_preferred` |
| `Country` | STRING | Should be `country_code CHAR(3)` ISO 3166-1 |
| `Business Center Name` | STRING | Denormalized; should be FK `business_center_bk` to `silver.org` |
| `E-statement delivery indicator` | STRING | Should be `BOOLEAN is_estatement_enabled` |

**Mapping**: 15/30 (50%). Entity empty in production.

#### 4.3.3 Metadata Completeness
Identical deficiencies to §4.1.3. Physical table empty.

---

### 4.4 Entity: `Customer_Demographics`

#### 4.4.1 Industry Fitness

**Boundary**: Individual customer demographics — nationality, income, employment, marital status, education. Scope is mostly appropriate. Issues:
- `US Relation` and `TIN Number` are FATCA/CRS attributes — arguable overlap with `silver.risk`.
- `Monthly Income` and `Annual income` both exist here, and `Gross Monthly Income` also exists in the parent `Customer` entity — income data is duplicated/triplicated across entities.
- `Residing Country` duplicates `Country of domicile` in the parent `Customer` entity.
- No discriminator on whether this row is for an individual customer (corporate customers should not have demographics records).

**Physical table**: `customer_demographics` — deployed with data (106 MB, 3 files). One of only 2 core entities with actual data.

#### 4.4.2 Attribute-Level Review

| Attribute | LM Type | Issue |
|---|---|---|
| `Monthly Income` | STRING | Should be `DECIMAL(18,4)` or `BIGINT`; duplicates `Gross Monthly Income` in Customer |
| `Annual income` | STRING | Should be `DECIMAL(18,4)` or `BIGINT`; approximate duplicate |
| `Number of dependents` | STRING | Should be `INTEGER` |
| `Gender code` | STRING | Canonicalize via `silver.reference.code_mapping` |
| `Marital status` | STRING | Canonicalize via reference |
| `Education code` | STRING | Canonicalize via reference |
| `Ethnic code` | STRING | Sensitive PII; must be access-controlled (column-level security, Guideline §3.10) |
| `Expiry Date` | STRING | Should be `DATE`; expiry of what? — ambiguous definition |
| `Employed Date` | STRING | Should be `DATE` |
| `TIN Number` | STRING | PII — SHA-256 masking; FATCA field; triplicated across entities |
| `US Relation` | STRING | FATCA; should be `BOOLEAN is_us_related`; belongs in `silver.risk` |
| `Country of Birth` | STRING | Should be `country_code CHAR(3)` |
| `Country of Tax Residence` | STRING | Should be `country_code CHAR(3)` |
| `Residing Country` | STRING | Duplicate of `Country of domicile` in Customer |
| `Current employment` | STRING | Should be `BOOLEAN is_employed` |
| `Current employment tenure` | STRING | Should be `INTEGER` (months) |
| `Occupation code permanent resident indicator` | STRING | Non-snake_case; overly long name; should be `occupation_permanent_resident_ind` |
| `Own home Indicator` | STRING | Should be `BOOLEAN is_own_home` |
| `Signed Date` | STRING | Should be `DATE`; context unclear |

**Mapping**: 8/36 (22%).

#### 4.4.3 Metadata Completeness
Identical deficiencies to §4.1.3.

---

### 4.5 Entity: `Customer_Documents`

#### 4.5.1 Industry Fitness

**Boundary**: Identity document registry — EID, Passport, Trade License, CRS undocumented flag, document verification. Mostly appropriate scope.

Issues:
- `Passport Number` and `EID Number` are critical PII fields. DQ Controls §2 (verbatim): *"EID, Card Numbers, Phone Numbers, and Passport Numbers masked via SHA-256 unless stored in a restricted container."* Both are currently unmapped with no masking directive — a regulatory P0 gap.
- `ID number` / `ID type` in the parent `Customer` entity duplicates this data.
- `Document Code Description` and `Document Type Description` are derivable from code lookups — storing both code and description in Silver violates Guideline §2.6 (derived values).
- `Documents Collected` in this entity duplicates the same attribute in `Customer_Preferences`.

**Physical table**: `customer_documents` — deployed with data (35 MB, 1 file).

#### 4.5.2 Attribute-Level Review

| Attribute | LM Type | Issue |
|---|---|---|
| `Passport Number` | STRING | **P0 — SHA-256 masking mandatory** per DQ §2; PII |
| `EID Number` | STRING | **P0 — SHA-256 masking mandatory** per DQ §2; PII |
| `Issue Date` | STRING | Should be `DATE` |
| `Document Type Description` | STRING | Derived from `Document Type` — Guideline §2.6 violation |
| `Document Code Description` | STRING | Derived from `Document Code` — Guideline §2.6 violation |
| `Is Document Verified?` | STRING | Question mark in column name violates NAM-001; should be `is_document_verified BOOLEAN` |
| `IDIssued Organization` | STRING | camelCase — NAM-001; rename `id_issuing_organization` |
| `Unique ID` | STRING | Ambiguous; should be `document_id` or `document_bk` |
| `Document Type` | STRING | Canonicalize via `silver.reference.code_mapping` |
| `Documents Collected` | STRING | Duplicate with `Customer_Preferences`; should be `BOOLEAN is_documents_collected` |
| `CRS Undocumented Flag` | STRING | Should be `BOOLEAN is_crs_undocumented`; also appears in `Customer_Relationships` |
| `Trade License Number` | STRING | PII-adjacent; masking decision needed |

**Mapping**: 12/28 (43%).

#### 4.5.3 Metadata Completeness
Identical deficiencies to §4.1.3.

---

### 4.6 Entity: `Customer_Groupings`

#### 4.6.1 Industry Fitness

**Boundary**: Customer group membership (holding groups, related-party groups). Appropriate scope. Issue: `Shareholding Pcnt.` is a percentage metric — Guideline §2.6 / rule SLV-007: *"Pre-compute totals, averages, counts, ratios, or any derived metrics"* must not appear in Silver. Raw shareholding numerator and denominator should be stored; percentage computed in Gold/Semantic.

**Physical table**: None deployed.

#### 4.6.2 Attribute-Level Review

| Attribute | LM Type | Issue |
|---|---|---|
| `Shareholding Pcnt.` | STRING | SLV-007 violation — percentage metric; should be `DECIMAL(18,4)` at minimum; store components instead |
| `Primary Grp Indicator` | STRING | `is_primary_group BOOLEAN`; "Grp" is non-approved abbreviation (NAM-001) |
| `Group ID` | STRING | Should be `group_bk` per NAM-001 |
| `Group Code` | STRING | Canonicalize via reference |

**Mapping**: 4/16 (25%). Not materialized.

#### 4.6.3 Metadata Completeness
Identical deficiencies to §4.1.3. No physical table.

---

### 4.7 Entity: `Customer_Preferences`

#### 4.7.1 Industry Fitness

**Boundary**: Communication preferences, consent flags (Do Not Call/SMS/Mail/Email), language preference, FATCA/CRS classification, trade license waivers. Severely over-scoped — mixes three unrelated concern categories:
1. **Communication preferences** (Do Not Call, Language): correct here.
2. **FATCA/CRS compliance attributes** (`US Relation`, `TIN Number`, `GIIN Number`, `Financial Entity`, `Fatca Entity Type`, `Exempted U S Entity`): belong in `silver.risk`.
3. **Trade license operational flags** (`Trade License Alert Waiver`, `Ledger Fee Waiver`): belong in `silver.account` or a product SA.

Mapping completeness of only 6% (3/51 attributes) indicates this entity is substantially undeveloped.

Critical naming violation: attribute named `Type` — this is a SQL reserved word, violating NAM-005: *"Never use SQL reserved words as identifiers."*

Attribute `Do Not Fax indicator in National registry Standard Industrial Classification code` appears to conflate two entirely unrelated concepts in a single attribute name — a data model defect, not merely a naming issue.

Attribute `Preference of Language` duplicates `Customer Preferred Language` — same entity, two names for the same concept.

**Physical table**: `customer_preferences` — empty (0 bytes).

#### 4.7.2 Attribute-Level Review

| Attribute | LM Type | Issue |
|---|---|---|
| `Type` | STRING | **NAM-005 VIOLATION** — SQL reserved word |
| `Do Not Fax indicator in National registry Standard Industrial Classification code` | STRING | Attribute name conflates fax indicator with SIC code — data model defect |
| `Preference of Language` | STRING | Duplicate of `Customer Preferred Language` |
| `Preferred Mobile Number` | STRING | PII — SHA-256 masking required |
| `Preferred Email id` | STRING | PII — masking; rename `preferred_email_address` |
| `TIN Number` | STRING | PII; FATCA; triplicated across entities; belongs in risk SA |
| `US Relation` | STRING | FATCA; `is_us_related BOOLEAN`; belongs in risk SA; duplicated from Demographics |
| `GIIN Number` | STRING | FATCA; belongs in risk SA |
| `Fatca Entity Type` | STRING | `fatca_entity_type_code`; belongs in risk SA |
| `Controlling person have U S Relation` | STRING | FATCA; belongs in risk SA |
| `Exempted U S Entity` | STRING | `is_us_entity_exempt BOOLEAN`; belongs in risk SA |
| `Financial Entity` | STRING | FATCA scope |
| `Litigations If any` | STRING | `has_litigation BOOLEAN`; "If any" is invalid name text |
| `Ledger Fee Waiver Special Condition` | STRING | Belongs in account SA |
| `Documents Collected` | STRING | Duplicate with `Customer_Documents.Documents Collected` |
| `Countries Set` | STRING | AML scope; belongs in risk SA |
| `Dealing with Countries` | STRING | AML scope; belongs in risk SA |
| `Name of security Market` | STRING | Belongs in wealth SA |
| `Name of Traded corporation` | STRING | Belongs in wealth/product SA |
| `Signed Date` | STRING | Should be `DATE` |
| `Expiry Date` | STRING | Should be `DATE` |
| `Do Not Call indicator in Bank registry` | STRING | `is_do_not_call_bank BOOLEAN` |
| `Do Not SMS indicator in Bank registry` | STRING | `is_do_not_sms_bank BOOLEAN` |
| `Category` | STRING | Ambiguous — which category? Definition absent |

**Mapping**: 3/51 (6%). Critical. Physical table empty.

#### 4.7.3 Metadata Completeness
Identical deficiencies to §4.1.3. Physical table empty.

---

### 4.8 Entity: `Customer_Psychographics`

#### 4.8.1 Industry Fitness

**Boundary**: KYC profile data — KYC review dates, expected transaction volumes, PEP status, DNFBP, sanctions declaration. **Severely misnamed**: "Psychographics" is a marketing analytics term (lifestyle, attitudes, values). The content is KYC/AML compliance data.

Per the SA taxonomy, `silver.risk` (SA #17, Priority 1 Critical) is designated for: *"KYC Levels, AML Risk Scores, Watchlist Hits, Customer Risk Ratings, FATCA/CRS flags."* This entity belongs there, not in `silver.customer`.

Two turnover percentage attributes (`Expected Monthly Cash Credit Turnover Percentage`, `Expected Monthly Non Cash Credit Turnover Percentage`) are pre-computed metrics — Guideline §2.6 / SLV-007 violation.

**Physical table**: None deployed.

#### 4.8.2 Attribute-Level Review

| Attribute | LM Type | Issue |
|---|---|---|
| `Watch list indicator` | STRING | `is_watchlisted BOOLEAN`; belongs in `silver.risk` |
| `KYC Held` | STRING | `is_kyc_held BOOLEAN`; belongs in `silver.risk` |
| `KYC Review Date` | STRING | Should be `DATE`; belongs in `silver.risk` |
| `PEP` | STRING | `is_pep BOOLEAN`; belongs in `silver.risk`; abbreviation not in approved list |
| `Sanction Declaration Status` | STRING | Belongs in `silver.risk`; canonicalize code |
| `DNFBP Declaration Status` | STRING | Belongs in `silver.risk` |
| `Expected Monthly Cash Credit Turnover Percentage` | STRING | **SLV-007 violation** — percentage metric; should be `DECIMAL(18,4)` |
| `Expected Monthly Non Cash Credit Turnover Percentage` | STRING | **SLV-007 violation** |
| `Expected Monthly Credit Turnover Amount` | STRING | Should be `BIGINT` (fils) or `DECIMAL(18,4)` |
| `Expected Highest Cash Credit Transaction Amount` | STRING | Should be `BIGINT` (fils) |
| `No of Permanent Employees` | STRING | Should be `INTEGER`; belongs in `Customer_Business` entity |

**Mapping**: 14/26 (54%). Entity misscoped.

#### 4.8.3 Metadata Completeness
Identical deficiencies to §4.1.3. No physical table.

---

### 4.9 Entity: `Customer_Relationships`

#### 4.9.1 Industry Fitness

**Boundary**: Related-party relationships (UBO, guarantor, authorized signatory, controlling person, household member). Appropriate scope in principle. Issues:
- `Relationship Type` — the essential discriminator classifying what kind of relationship this is — has mapping status `N` (unmapped). Without this, the entity carries data but cannot be semantically interpreted.
- The entity embeds a mini-party profile for the related party (`Full Name`, `Nationality Code`, `Date of Birth / Date of Incorporation`, `Address Type`, `Place of Birth`) instead of referencing the related party via FK. This creates data duplication if the related party is themselves also a bank customer.
- `Total Liability Pcnt.` and `Total Equity Pcnt.` and `% of controlling interest` are all percentage metrics — SLV-007 violations.
- `Date of Birth / Date of Incorporation` conflates two distinct data types in one column — individual DoB vs. corporate incorporation date. Must be split.
- `TIN Number`, `US Relation`, `CRS Undocumented Flag` appear here for the third time across entities — uncontrolled duplication.
- `Passport No./ Trade License` — mixed types in one column; also PII requiring SHA-256 masking.

**Physical table**: `customer_relationships` — deployed but empty (0 bytes).

#### 4.9.2 Attribute-Level Review

| Attribute | LM Type | Issue |
|---|---|---|
| `Relationship Type` | STRING | **Critical unmapped** — discriminator for the entire entity |
| `Total Liability Pcnt.` | STRING | **SLV-007 violation** — percentage metric; `pct` not spelled out; `DECIMAL` type |
| `Total Equity Pcnt.` | STRING | **SLV-007 violation** |
| `% of controlling interest` | STRING | **SLV-007 violation** — rename `controlling_interest_pct DECIMAL(18,4)` |
| `Date of Birth/ Date of Incorporation` | STRING | Should be `DATE`; conflates two types — split into `date_of_birth` and `date_of_incorporation` |
| `Passport No./ Trade License` | STRING | Mixed types in one column; PII — **SHA-256 masking mandatory** |
| `CIF ID` | STRING | Rename `related_customer_bk` per NAM-001 |
| `Related CIF ID` | STRING | Potential duplicate of `CIF ID`; clarify semantics |
| `Name` | STRING | Generic name violates NAM; PII; rename `related_party_name` |
| `Nationality` | STRING | Derived from `Nationality Code` — SLV-006 violation; remove description, keep code |
| `Country of Residence` | STRING | Derived from `Country of Residence Code` — SLV-006 violation |
| `TIN Number` | STRING | Triplicated across entities; PII; FATCA; belongs in risk SA |
| `US Relation` | STRING | FATCA; triplicated; belongs in risk SA |
| `IB Enabled Flag` | STRING | `is_ib_enabled BOOLEAN` |
| `CRS Undocumented Flag` | STRING | `BOOLEAN`; duplicate with `Customer_Documents` |

**Mapping**: 9/40 (23%). Physical table empty.

#### 4.9.3 Metadata Completeness
Identical deficiencies to §4.1.3. Physical table empty.

---

### 4.10 Entity: `Customer_Individual`

#### 4.10.1 Industry Fitness

**Boundary**: Intended as a sub-type of `Customer` for individual (retail) customers per Party/sub-type pattern. Currently an empty shell — 5 attributes total, all of which are composite PK fields or SCD-2 audit columns. No individual-specific business attributes have been populated. All individual attributes (DoB, gender, marital status, employment) are scattered in the parent `Customer` entity and `Customer_Demographics`.

**Physical table**: None deployed. Zero mapping rows.

#### 4.10.2 Attribute-Level Review

| Attribute | LM Type | Issue |
|---|---|---|
| `Customer number (CIF)` | STRING | FK to parent; not `customer_bk` |
| `Customer Type` | STRING | Sub-type discriminator; should be constrained to individual type values |
| `Business Start Date` | CHAR(18) | `effective_from TIMESTAMP NOT NULL` |
| `Business End Date` | CHAR(18) | `effective_to TIMESTAMP` |
| `Is Active Flag` | CHAR(18) | `is_active_flag STRING` |

**Mapping**: 0/5 (0%). Entity is a governance placeholder with no content.

#### 4.10.3 Metadata Completeness
All required metadata absent. No physical table.

---

### 4.11 Entity: `Customer_Business`

#### 4.11.1 Industry Fitness

**Boundary**: Intended as a sub-type of `Customer` for corporate/business customers. Identical structural problem to `Customer_Individual` — 5 attributes, all PK/audit columns, no business-specific content. Corporate attributes (Trade License, SIC codes, registered address, company type, date of incorporation) are scattered across `Customer`, `Customer_ContactInformation`, `Customer_Documents`, and `Customer_Groupings`. The sub-type pattern is broken.

**Physical table**: None deployed. Zero mapping rows.

#### 4.11.2 Attribute-Level Review

Identical pattern to §4.10.2. All 5 attributes are PK/SCD placeholders. No business content.

**Mapping**: 0/5 (0%). Entity is a governance placeholder.

#### 4.11.3 Metadata Completeness
All required metadata absent. No physical table.

---

### 4.12 Reference Entities (20 entities — bulk assessment)

All 20 `Reference_*` entities share the same structural deficiencies. Per-entity highlights are provided in the table; the common deficiency block covers all 20.

#### Common Deficiencies Across All 20 Reference Entities

| Deficiency | Detail |
|---|---|
| **Wrong SA** | All 20 belong in `silver.reference` per SA taxonomy (SA-002) |
| **No SK / BK** | Zero `<entity>_sk` or `<entity>_bk` attributes anywhere |
| **No `effective_from` / `effective_to` / `is_current`** | SCD columns absent (SCD-001) |
| **No `run_id` / `source_ingestion_date`** | Missing from all (SLV-008) |
| **All attributes STRING** | Code values, dates, indicators — all typed STRING |
| **Missing physical tables** | 17 of 20 have no physical Delta table |
| **`Reference_Occupation_PermanentResident`** | Only 2 attributes — missing `source_system_code`, `source_system_id`, all SCD/audit columns. Incomplete model. |

#### Per-Reference Entity Summary

| Entity | LM Attrs | Physical Table | Mapped % | Critical Gap |
|---|---|---|---|---|
| Reference_SIC_Code | 9 | ❌ None | 0% | No physical; 0% mapped |
| Reference_SIC_Group | 8 | ❌ None | 0% | No physical; 0% mapped |
| Reference_SIC_EconomicMeasure | 11 | ❌ None | 0% | No physical; 0% mapped |
| Reference_Business_SIC_History | 11 | ❌ None | 0% | No physical; 0% mapped |
| Reference_Customer_Status | 8 | ❌ None | 100% | No physical despite 100% mapping |
| Reference_Customer_Type | 8 | ✅ (wrong SA) | 100% | Wrong SA; auto-optimize missing |
| Reference_Country | 16 | ✅ (wrong SA) | 40% | Wrong SA; 60% unmapped; column `Deleted Date` inconsistent with `Delete Date` elsewhere |
| Reference_Ethnicity | 9 | ❌ None | 0% | Sensitive PII; no physical |
| Reference_Gender | 10 | ❌ None | 100% | No physical; `Deleted Date` naming inconsistency |
| Reference_Organization_Type | 8 | ❌ None | 100% | No physical |
| Reference_ID_Type | 9 | ❌ None | 100% | No physical; `Deleted Date` inconsistency |
| Reference_Education | 8 | ❌ None | 100% | No physical; `Deleted Date` inconsistency |
| Reference_Own_Home | 8 | ❌ None | 0% | No physical; 0% mapped |
| Reference_Marital_Status | 8 | ❌ None | 100% | No physical |
| Reference_Occupation_PermanentResident | 2 | ❌ None | 0% | Incomplete model — 2 attributes only, all audit columns absent |
| Reference_Residence_Type | 8 | ❌ None | 100% | No physical |
| Reference_Segment | 9 | ✅ empty (wrong SA) | 100% | Wrong SA; table empty |
| Reference_Customer_Business_Classification | 8 | ❌ None | 100% | No physical |
| Reference_Relationship_Manager | 11 | ❌ None | 20% | Should be in `silver.org`, not reference; 80% unmapped |
| Reference_Credit_Grade | 8 | ❌ None | 100% | No physical |

**Additional finding on `Reference_Country`**: Column named `Deleted Date` while all other entities use `Delete Date`. Inconsistent naming across the same LM violates NAM-001.

**Additional finding on multiple Reference entities**: Several entities use `Deleted Date` (past tense) instead of `Delete Date` (consistent with Guideline §2.7 `delete_date`). This inconsistency affects `Reference_Country`, `Reference_Gender`, `Reference_ID_Type`, `Reference_Education`, `Reference_Segment`.

---

## 5. Task 4 — Denormalization Analysis

### 5.1 Denormalization Register

| Attribute(s) | Host Entity | Should Originate From | Classification | Data-Loss Risk | Remediation |
|---|---|---|---|---|---|
| `Primary RM Code/Name/Mobile/Email` + `Secondary RM *` + `Tertiary RM *` (12 attrs) | `Customer` | `Reference_Relationship_Manager` / `silver.org` | ❌ **Unjustified repeating group** — 3NF §2.1 violation | HIGH: 4th RM assignment silently dropped | Normalize to `customer_rm_assignment` child table |
| `Primary Call Back Contact`, `Primary Call Back Contact Number`, `Secondary Call Back Contact`, `Secondary Call Back Contact Number` (4 attrs) | `Customer` | `Customer_ContactInformation` | ❌ **Unjustified repeating group** | MEDIUM: 3rd call-back contact dropped | Move to `customer_contact` child table |
| `Gross Monthly Income` | `Customer` | `Customer_Demographics` | ❌ **Unnecessary duplication** | MEDIUM: conflicting values if updated in one entity | Remove from `Customer`; single source in Demographics |
| `Monthly Income` | `Customer_Demographics` | `Customer` | ❌ **Duplication** | MEDIUM | Consolidate to one entity |
| `Annual income` | `Customer_Demographics` | — | ❌ **Near-duplicate** of monthly income | MEDIUM | Decide canonical income attribute; remove duplicate |
| `Industry Type`, `Sector`, `Sub Sector`, `SIC Code` | `Customer` | `Reference_SIC_Code` / `Reference_SIC_Group` | ⚠️ **Acceptable if FK documented** | LOW | Add FK to Reference_SIC_Code; document denormalization justification |
| `Nationality` | `Customer_Relationships` | `Reference_Country` via `Nationality Code` | ❌ **Derived value** — Guideline §2.6 | LOW: description drift if reference changes | Remove `Nationality`; keep `Nationality Code` only |
| `Country of Residence` | `Customer_Relationships` | `Reference_Country` via `Country of Residence Code` | ❌ **Derived value** — Guideline §2.6 | LOW | Remove description; keep code |
| `Document Type Description` | `Customer_Documents` | Reference lookup | ❌ **Derived value** — Guideline §2.6 | LOW | Remove description column |
| `Document Code Description` | `Customer_Documents` | Reference lookup | ❌ **Derived value** — Guideline §2.6 | LOW | Remove description column |
| `TIN Number` | Demographics + Preferences + Relationships | One canonical location | ❌ **Triplication** — same PII in 3 entities | HIGH: inconsistent masking, inconsistent values | Consolidate to `Customer_Documents`; remove from others |
| `US Relation` | Demographics + Preferences + Relationships | One canonical location (risk SA) | ❌ **Triplication** | HIGH: conflicting FATCA status across entities | Consolidate to `silver.risk` |
| `CRS Undocumented Flag` | Documents + Relationships | One canonical location | ❌ **Duplication** | MEDIUM | Consolidate to `Customer_Documents` |
| `Business Center Name` | `Customer_ContactInformation` | `silver.org` | ❌ **Unjustified denormalization** | MEDIUM: name drift if org changes | Replace with `business_center_bk` FK |
| `Enable E-Banking`, `SMS Banking`, `WAP Banking`, `Phone Banking` | `Customer` | `Customer_Preferences` | ❌ **Unnecessary duplication** | MEDIUM: conflicting preference values | Remove from `Customer`; sole source in Preferences |
| `Documents Collected` | Documents + Preferences | One canonical location | ❌ **Duplication** | LOW | Consolidate to `Customer_Documents` |

### 5.2 Structure Extensibility Assessment

| Question | Assessment |
|---|---|
| Can the model accommodate a 4th RM tier? | ❌ No — requires DDL change (new columns). Normalized child table would require only a new data row. |
| Can the model accommodate a 5th address? | ❌ No — `address1–4` fixed-width; 5th address silently dropped. |
| Can the model represent an individual who is both retail and private banking customer? | ❌ No — one row per CIF; dual-role representation impossible. |
| Can the model store multiple phone numbers with type/verification? | ❌ No — single `Phone number` column. |
| Will it silently drop data from source? | ✅ **YES** — for any source that provides >4 address lines, >3 RM tiers, >1 phone/email, all surplus values are silently dropped. |

### 5.3 Fit-for-Industry Pattern Assessment

| Pattern | Status |
|---|---|
| Party / Role separation (BIAN / IFW) | ❌ Not implemented — Party identity and CustomerRole mixed |
| Party / Address / Identifier / Contact sub-tables (IFW) | ❌ Not implemented — contact is a flat single-row entity with numbered columns |
| SCD-2 change history | ❌ Not implementable — columns misnamed/mistyped, `appendOnly` blocks MERGE |
| MD5 deterministic surrogate key | ❌ Absent on all 31 entities |
| Code canonicalization via reference mapping | ❌ Straight-through for all code fields |
| PII masking at Silver gate | ❌ No masking directives in logical model or mapping |
| Consent management entity | ❌ Absent — consent scattered as flags across multiple entities |

---

## 6. Guideline Compliance Summary

| Rule | Description | Status | Affected Entities | Finding Ref |
|---|---|---|---|---|
| **Guideline §2.1** | 3NF — no repeating groups or numbered columns | ❌ FAIL | Customer_ContactInformation, Customer | address1–4 pattern; RM tier repeating group |
| **Guideline §2.3 / SLV-003** | SCD-2 default; `effective_from`, `effective_to`, `is_current` on every entity | ❌ FAIL | All 31 | SCD-001: wrong names, wrong types, `appendOnly` blocks MERGE |
| **Guideline §2.4 / SLV-001** | MD5 deterministic surrogate key `<entity>_sk` | ❌ FAIL | All 31 | ID-001: zero surrogate keys in entire model |
| **Guideline §2.4 / SLV-002** | Business key `<entity>_bk` on every entity | ❌ FAIL | All 31 | ID-002: zero `_bk` columns in entire model |
| **Guideline §2.5 / SLV-004** | No raw source codes; canonicalize via `silver.reference.code_mapping` | ❌ FAIL | Customer, Demographics, Documents, Relationships | ETL-002: straight-through mapping |
| **Guideline §2.6 / SLV-006** | No derived values in Silver | ❌ FAIL | Customer (Full Name), Documents (descriptions), Relationships (descriptions) | Guideline §2.6 |
| **Guideline §2.6 / SLV-007** | No pre-computed metrics / percentages | ❌ FAIL | Psychographics, Groupings, Relationships | Shareholding Pcnt., Liability Pcnt., turnover pct. |
| **Guideline §2.7 / SLV-008** | 11 mandatory Silver audit columns present | ❌ FAIL | All 31 | `run_id` and `source_ingestion_date` absent; others misnamed/mistyped |
| **Guideline §2.9** | All timestamps in UTC | ⚠️ UNVERIFIABLE | All | All date/time columns typed STRING or CHAR(18); UTC enforcement cannot be assessed |
| **Guideline §2.10** | Multi-source: companion tables, not wide table | ❌ FAIL | Customer | Finacle + RLS Loan merged into single entity; ETL-003 |
| **Guideline §2.11** | Erwin model before DDL | ❌ FAIL | `credit_limit_liability` | Physical table with no logical entity |
| **Guideline §2.12** | Data catalog registration | ⚠️ PARTIAL | 9 of 10 tables | Only `customer` has a catalog description |
| **NAM-001** | snake_case, lowercase identifiers | ❌ FAIL | All 31 | All LM attribute names are Title Case with spaces; camelCase on metadata cols |
| **NAM-002** | Catalog: `silver.<subject_area>.<entity>` | ❌ FAIL | All 10 tables | Production catalog is `silver_dev_v2` not `silver` |
| **NAM-003** | Table: `<entity>` for Silver | ⚠️ PARTIAL | `customer_contactinformation` | Missing underscore; should be `customer_contact_information` |
| **NAM-005** | No SQL reserved words as identifiers | ❌ FAIL | Customer_Preferences | Attribute named `Type` |
| **DQ §2 / DQ-003** | PII masking: EID, Passport, Phone, TIN via SHA-256 | ❌ FAIL | Documents, ContactInformation, Demographics, Relationships, Preferences | No masking directives on any PII field |
| **DQ §3.1 / DQ-001** | Deduplication on business keys | ⚠️ UNVERIFIABLE | All | No `record_hash`; dedup pipeline undocumented |
| **DQ §4 / DQ-002** | Monetary columns `DECIMAL(18,4)` | ❌ FAIL | Customer, Demographics, AdditionalFinancial, Psychographics | All monetary amounts typed STRING |
| **DQ §5** | 11 mandatory DQ audit columns | ❌ FAIL | All 11 core | `record_source`, `record_origin`, `record_hash`, `batch_id`, `data_quality_status`, `dq_issues` all absent |
| **DQ §10 / PDPL** | Right-to-be-forgotten pipeline within 30 days | ❌ FAIL | All PII-bearing entities | No consent record; no erasure pipeline possible without structured consent entity |

---

## 7. Remediation Plan

### 7.1 P0 — Immediate (Current Sprint)

| # | Finding | Action | Owner |
|---|---|---|---|
| P0-01 | SA-001 | Convene architecture board: decide (a) merge `silver.customer` into `silver.party` or (b) formally register `customer` SA in `sa/subject-areas.md` with schema and steward. No downstream work is safe until this decision is made. | Data Steward + Architecture Board |
| P0-02 | ID-001/002 | Add `customer_sk STRING NOT NULL` (`= MD5(UPPER(TRIM(customer_bk)))`) and `customer_bk STRING NOT NULL` to `Customer` entity. Add `<entity>_sk` and `<entity>_bk` to all 31 entities. Propagate `customer_bk` as named FK on all child entities. | Data Modeler |
| P0-03 | SCD-001 | Rename `Business Start Date` → `effective_from TIMESTAMP NOT NULL`. Rename `Business End Date` → `effective_to TIMESTAMP`. Replace `IsLogicallyDeleted STRING` → `is_current BOOLEAN NOT NULL`. Apply to all 11 core entities. | Data Modeler + DBA |
| P0-04 | SCD-001 / INF-003 | Remove `appendOnly` Delta feature from all 9 affected tables to enable SCD-2 MERGE operations. | Data Engineer |
| P0-05 | DQ-003 | Immediately implement SHA-256 masking for: `Passport Number` (Customer_Documents), `EID Number` (Customer_Documents), `Mobile Number` (Customer_ContactInformation), `Preferred Mobile Number` (Customer_Preferences), `TIN Number` (all 3 entities), `Passport No./ Trade License` (Customer_Relationships). PDPL/CBUAE KYC compliance. | Data Security + Pipeline Engineer |
| P0-06 | ETL-001 | Develop source-to-target mapping for all unmapped attributes: complete `Customer_Individual`, `Customer_Business`, and the 90+ unmapped attributes in `Customer` entity. | Mapping Analyst |
| P0-07 | ETL-002 | Intercept all "straight-through" code field mappings and route through `silver.reference.code_mapping`. Minimum scope: Customer Type, Customer Status, Gender, Marital Status, Industry Type, Document Type, Country codes. | Pipeline Engineer |
| P0-08 | SLV-008 | Add missing audit columns to all entity logical model definitions: `run_id STRING`, `source_ingestion_date TIMESTAMP`. | Data Modeler |
| P0-09 | DQ §5 | Add DQ metadata columns to all entity LM definitions: `record_hash STRING`, `batch_id STRING`, `data_quality_status STRING`, `dq_issues STRING`. | Data Modeler |
| P0-10 | Mapping quality | Fix mapping document: rename column `Enitity_Name` → `Entity_Name`; fix entity `Customer_Relatioships` → `Customer_Relationships`; fix `Souce_System.Column_Data_Type` → `Source_System.Column_Data_Type`. | Mapping Analyst |

### 7.2 P1 — Next Sprint

| # | Finding | Action | Owner |
|---|---|---|---|
| P1-01 | SA-002 | Migrate 3 physical reference tables (`reference_country`, `reference_customer_type`, `reference_segment`) from `silver_dev_v2.customer` to `silver.reference`. Update all pipeline references. Remove all 20 reference entities from customer SA logical model. | Data Engineer |
| P1-02 | SA-003 | Investigate `credit_limit_liability`: model it in LM or migrate to `silver.account`/`silver.lending`. | Data Steward |
| P1-03 | RI-001 | Define FK relationships in logical model for all child entities → `Customer`. Add referential integrity DQ checks to Silver pipeline. | Data Modeler |
| P1-04 | 3NF | Normalize `Customer_ContactInformation`: replace `mail_address1–4` and `registered_address1–4` with new `customer_address` child table. | Data Modeler |
| P1-05 | Missing entity | Create `customer_consent` entity for structured PDPL consent management. | Data Modeler |
| P1-06 | Missing entity | Create `customer_contact` child table to replace single-row phone/email columns. | Data Modeler |
| P1-07 | Denorm | Create `customer_rm_assignment` child table to replace 12 RM-tier repeating columns in `Customer`. | Data Modeler |
| P1-08 | Types | Fix all data types in logical model: dates → `DATE`; timestamps → `TIMESTAMP`; monetary amounts → `BIGINT` (fils) or `DECIMAL(18,4)`; boolean indicators → `BOOLEAN`; codes → `STRING` (mark for canonicalization). | Data Modeler |
| P1-09 | SLV-007 | Remove all percentage metrics from Silver: `Shareholding Pcnt.` from Groupings; `Total Liability Pcnt.`, `Total Equity Pcnt.`, `% of controlling interest` from Relationships; turnover percentages from Psychographics. Store raw additive components only. | Data Modeler |
| P1-10 | ETL-003 | Document multi-source conflict resolution for Finacle + RLS Loan attributes. Implement companion tables per Guideline §2.10 or formally document precedence rules with steward sign-off. | Pipeline Architect |
| P1-11 | INF-001 | Add partition column `_ingested_date DATE` to all 10 tables and rebuild. Consider `effective_from::date` as secondary partition for SCD-2 tables. | Data Engineer |
| P1-12 | INF-002 | Enable `delta.autoOptimize.autoCompact=true` and `delta.autoOptimize.optimizeWrite=true` on 7 tables lacking it. | Data Engineer |
| P1-13 | Scope | Move `Customer_Psychographics` (KYC/AML data) to `silver.risk` SA. Coordinate with Risk SA data steward. | Architecture + Data Steward |
| P1-14 | Catalog | Load data into empty tables: `customer_contactinformation`, `customer_preferences`, `customer_relationships`, `reference_segment`. | Pipeline Engineer |
| P1-15 | NAM-001 | Rename all logical model attribute names from Title Case with spaces to snake_case: e.g., `Customer number (CIF)` → `customer_bk`, `CIF Creation Date` → `cif_creation_date`. Update mapping document to match. | Data Modeler |

### 7.3 P2 — Backlog

| # | Finding | Action |
|---|---|---|
| P2-01 | SCD-002 | Document SCD strategy per entity. Obtain data steward approval for SCD-1 exceptions (reference tables). |
| P2-02 | Denorm-TIN | Consolidate `TIN Number` to one canonical location (`Customer_Documents`). Remove from Preferences and Relationships. |
| P2-03 | NAM-005 | Rename `Type` in `Customer_Preferences` (SQL reserved word). Propose `fatca_type_code`. |
| P2-04 | SLV-006 | Remove `Customer Full Name` as a concatenated derived column. Store `first_name`, `middle_name`, `last_name` separately; compute full name in Gold or Semantic. |
| P2-05 | Sub-types | Populate `Customer_Individual` and `Customer_Business` with their respective discriminating attributes. Move individual-specific attributes from `Customer` to `Customer_Individual`; corporate-specific to `Customer_Business`. |
| P2-06 | Catalog | Register all Silver tables in Informatica GDGC with data classification, steward name, subject area, and lineage links per Guideline §2.12. |
| P2-07 | Access | Audit Unity Catalog access controls: ensure no analyst/BI service accounts have direct SELECT on `silver_dev_v2.customer.*` for PII-bearing tables (`customer`, `customer_documents`, `customer_demographics`, `customer_relationships`). |
| P2-08 | SLV-006 | Remove `Document Type Description` and `Document Code Description` from `Customer_Documents`. Remove derived description columns from `Customer_Relationships` (`Nationality`, `Country of Residence`). |
| P2-09 | Maintenance | Set up weekly `VACUUM` schedule on all Delta tables to prevent unbounded file version growth. |

### 7.4 Suggested Delivery Schedule

| Sprint | Focus | Key Deliverables |
|---|---|---|
| Sprint 1 (Wk 1–2) | Architecture + Identity | P0-01: SA decision; P0-02: SK/BK added; P0-05: PII masking |
| Sprint 2 (Wk 3–4) | SCD + Metadata | P0-03/04: SCD columns fixed, appendOnly removed; P0-08/09: audit columns added |
| Sprint 3 (Wk 5–6) | Mapping + Canonicalization | P0-06/07/10: mapping gap closed to 80%+; code canonicalization |
| Sprint 4 (Wk 7–8) | Normalization + New Entities | P1-04–07: address/contact/consent/RM child tables; 3NF fix |
| Sprint 5 (Wk 9–10) | Types + Infrastructure | P1-08–12: data types, partitioning, auto-optimize, loading |
| Sprint 6 (Wk 11–12) | Scoping + Governance | P1-01/13: reference SA migration; Psychographics to risk SA; catalog registration |
| Sprint 7+ (Backlog) | Polish | P2-01 through P2-09 |

---

## 8. Appendix

### Appendix A: Entity Inventory

| # | Entity | Physical Table | Has Data | SCD Cols | SK | BK | Metadata OK | P0 Findings |
|---|---|---|---|---|---|---|---|---|
| 1 | Customer | `customer` (244 MB) | ✅ | ❌ misnamed | ❌ | ❌ | ❌ | 8 |
| 2 | Customer_AdditionalFinancialInformation | None | ❌ | ❌ | ❌ | ❌ | ❌ | 3 |
| 3 | Customer_ContactInformation | `customer_contactinformation` (empty) | ❌ | ❌ | ❌ | ❌ | ❌ | 4 |
| 4 | Customer_Demographics | `customer_demographics` (106 MB) | ✅ | ❌ | ❌ | ❌ | ❌ | 3 |
| 5 | Customer_Documents | `customer_documents` (35 MB) | ✅ | ❌ | ❌ | ❌ | ❌ | 3 |
| 6 | Customer_Groupings | None | ❌ | ❌ | ❌ | ❌ | ❌ | 2 |
| 7 | Customer_Preferences | `customer_preferences` (empty) | ❌ | ❌ | ❌ | ❌ | ❌ | 3 |
| 8 | Customer_Psychographics | None | ❌ | ❌ | ❌ | ❌ | ❌ | 2 |
| 9 | Customer_Relationships | `customer_relationships` (empty) | ❌ | ❌ | ❌ | ❌ | ❌ | 3 |
| 10 | Customer_Individual | None | ❌ | ❌ | ❌ | ❌ | ❌ | 2 |
| 11 | Customer_Business | None | ❌ | ❌ | ❌ | ❌ | ❌ | 2 |
| 12–31 | Reference_* (20) | 3 of 20 (all wrong SA) | 1 of 20 | ❌ | ❌ | ❌ | ❌ | 1 each |

### Appendix B: Mapping Completeness by Entity

| Entity | LM Attrs | Mapped (Y) | Unmapped (N) | Coverage |
|---|---|---|---|---|
| Customer | 145 | 55 | 90 | 38% |
| Customer_AdditionalFinancialInformation | 25 | 3 | 22 | 12% |
| Customer_ContactInformation | 30 | 15 | 15 | 50% |
| Customer_Demographics | 36 | 8 | 28 | 22% |
| Customer_Documents | 28 | 12 | 16 | 43% |
| Customer_Groupings | 16 | 4 | 12 | 25% |
| Customer_Preferences | 51 | 3 | 48 | 6% |
| Customer_Psychographics | 26 | 14 | 12 | 54% |
| Customer_Relationships | 40 | 9 | 31 | 23% |
| Customer_Individual | 5 | 0 | 5 | 0% |
| Customer_Business | 5 | 0 | 5 | 0% |
| **Core Total** | **407** | **123** | **284** | **~30%** |

### Appendix C: Mapping Document Quality Issues

| Issue | Location | Impact |
|---|---|---|
| Column header typo `Enitity_Name` | Customer Silver Data Mapping, col A | Entity filtering breaks programmatic consumers |
| Entity name `Customer_Relatioships` | Customer Silver Data Mapping, rows for Relationships entity | FK join failures if referenced programmatically |
| Column header `Souce_System.Column_Data_Type` | Customer Silver Data Mapping, col I | Schema validation failures |
| Sheet named `Customer Silver Refinment Map` (typo) | Workbook tab | Low documentation maturity signal |
| `Status: Not Available` on multiple rows | Multiple rows | Mapping not completed |

### Appendix D: Guideline Citations

| Rule Code | Source Document | Exact Rule Text |
|---|---|---|
| Guideline §2.1 | Modeling DOs & DON'Ts | "DON'T: Add numbered or suffixed columns to represent multiple instances of the same concept (e.g., `address1`, `address2`, `address3`, `address4`)." |
| Guideline §2.3 | Modeling DOs & DON'Ts | "Every Silver entity table must have `effective_from`, `effective_to`, `is_current`." |
| Guideline §2.4 | Modeling DOs & DON'Ts | "Derive surrogate keys deterministically: `MD5(UPPER(TRIM(business_key)))` for single-component keys." |
| Guideline §2.5 | Modeling DOs & DON'Ts | "DO: Resolve all source-system codes to canonical platform values using `silver.reference.code_mapping`. DON'T: Store source-system-specific codes directly in Silver entity columns." |
| Guideline §2.6 | Modeling DOs & DON'Ts | "DO: Store only atomic, un-aggregated facts in Silver. DON'T: Pre-compute totals, averages, counts, ratios, or any derived metrics in Silver tables." |
| Guideline §2.7 | Modeling DOs & DON'Ts | "Include the following Silver technical audit columns on every Silver entity table: `source_system_code`, `source_system_id`, `create_date`, `update_date`, `delete_date`, `is_active_flag`, `effective_from`, `effective_to`, `is_current`, `run_id`, `source_ingestion_date`." |
| Guideline §2.9 | Modeling DOs & DON'Ts | "Store all timestamps in UTC. Convert source-local timestamps at ingestion." |
| Guideline §2.10 | Modeling DOs & DON'Ts | "When the same logical entity is sourced from multiple systems, create a canonical base entity table plus per-source companion tables." |
| Guideline §2.11 | Modeling DOs & DON'Ts | "Tables that exist in the platform but have no corresponding entity in the Erwin model are a governance gap." |
| NAM-001 | Naming Conventions | "Use snake_case for all identifiers. Use lowercase only — no mixed case. No abbreviations unless universally understood domain terms." |
| NAM-005 | Naming Conventions | "Never use SQL reserved words as identifiers (e.g., `date`, `value`, `order`)." |
| DQ §2 | DQ Controls | "PII Masking: EID, Card Numbers, Phone Numbers, and Passport Numbers masked via SHA-256 unless stored in a restricted container." |
| DQ §4 | DQ Controls | "All monetary columns must use `DECIMAL(18,4)`." |
| DQ §5 | DQ Controls | Mandatory audit columns: `record_source`, `record_origin`, `record_hash`, `is_current`, `valid_from_at`, `valid_to_at`, `ingested_at`, `processed_at`, `batch_id`, `data_quality_status`, `dq_issues`. |
| DQ §10 | DQ Controls | "Right to Be Forgotten: Automated pipelines must support erasure of customer data within 30 days." |
| SA Taxonomy | `sa/subject-areas.md` | "Customer is a Party role, not a separate SA — Avoids duplication of identity data; KYC and segmentation live in `silver.party`." |

### Appendix E: Industry Reference Standards Applied

| Standard | Version | Application |
|---|---|---|
| **BIAN** (Banking Industry Architecture Network) | v11 | Party Data Management domain used to evaluate entity separation; Customer Management domain for role-based attributes |
| **IFW** (IBM/Teradata Industry Framework for Banking) | Current | Party vs CustomerRole distinction; PartyRole sub-type pattern for Individual/Business sub-types; Party Address / Contact / Identifier decomposition |
| **BCBS 239** | 2013 | Principle 2 (Data Architecture) — SA scoping; Principle 6 (Completeness) — metadata and lineage columns |
| **UAE PDPL** | Federal Decree-Law No. 45 of 2021 | Article 5 — processing prohibition for opted-out customers; right-to-be-forgotten within 30 days; consent management entity requirement |
| **FATCA** | IRS Rev. Rul. 2014 | Multiple FATCA fields (`TIN Number`, `US Relation`, `GIIN`, `Fatca Entity Type`) found scattered across 3 entities — consolidation to `silver.risk` recommended |
| **CRS** (Common Reporting Standard) | OECD 2014 | `CRS Undocumented Flag` duplicated across Documents and Relationships — consolidation required |
| **3NF** (Codd) | Original | First/Third Normal Form violations: `address1–4` repeating groups; transitive dependencies (code + derived description in same entity) |
| **Kimball** | *The Data Warehouse Toolkit* 3rd ed. | Surrogate key pattern; conformed dimension placement for reference entities |
| **Databricks Delta Lake** | Current | `appendOnly` feature incompatibility with SCD-2; `changeDataFeed`, `columnMapping`, `rowTracking` features assessed |

---

*End of Report*
