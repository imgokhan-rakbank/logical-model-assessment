# Silver Layer Gap Assessment Report
## Subject Area: Application (`silver.application`)

**Assessment Date:** 2026-04-28  
**Assessed By:** Senior Data Platform Architect (Automated Assessment)  
**Assessment Version:** 1.0  
**Priority (per SA taxonomy):** 4 — Low  
**Schema:** `silver.application`  
**Source Artifacts:**
- Logical Model: `bank_logical_model.xlsx` → Sheet: `Consolidated Metadata for Subject` (28 entities, 409 rows)
- Physical Structures: `sa/application/application_physical_structures.csv` (49 table entries)
- Data Mapping Files: **None found** — no `*_data_mapping.xlsx` in `sa/application/input/`
- Guidelines: `guidelines/01-data-quality-controls.md`, `guidelines/06-naming-conventions.md`, `guidelines/11-modeling-dos-and-donts.md`

---

## 1. Executive Summary

### 1.1 Management Slide

The Application subject area (`silver.application`) was assessed against the full normative guideline set and compared against the logical model (28 entities) and physical structures (49 Delta tables). The assessment reveals **severe structural, governance, and modelling deficiencies** that render the current implementation materially non-compliant with the Silver-layer standards.

| Dimension | Status | Score |
|---|---|---|
| Architecture Coherence | 🔴 Critical | 2 / 10 |
| Entity Completeness | 🟡 Partial | 4 / 10 |
| Attribute Quality | 🔴 Critical | 2 / 10 |
| Key Strategy (SK/BK) | 🔴 Missing | 0 / 10 |
| SCD-2 Compliance | 🔴 Non-compliant | 2 / 10 |
| Metadata Completeness | 🟡 Partial | 3 / 10 |
| Naming Conventions | 🟡 Partial | 4 / 10 |
| Data Mapping Coverage | 🔴 Zero | 0 / 10 |
| Subject-Area Scoping | 🔴 Violated | 2 / 10 |
| DQ Controls | 🔴 Missing | 1 / 10 |
| **Overall** | 🔴 **Critical** | **2.0 / 10** |

**Critical headline findings:**
1. **No data mapping files exist** — ETL lineage is entirely undocumented for all 49 physical tables.
2. **No surrogate keys (`_sk`) or business keys (`_bk`)** are defined on any logical entity — a fundamental Silver layer identity requirement is absent across the entire subject area.
3. **`application_gold_fix` table** in Silver is a critical architecture violation — a Gold-workaround table stored in Silver with no logical model counterpart.
4. **`loan_gl_account`** belongs in `silver.gl`, not `silver.application` — SA contamination.
5. **`dummy_reference_education`** is a development artifact present in the Silver layer — critical governance failure.
6. **All logical attributes are typed as STRING or CHAR(18)** — no proper type specification anywhere in the model.
7. **22 of 28 logical entities are reference tables**, of which at least 12 should live in `silver.reference` or other canonical SAs, not duplicated in `silver.application`.
8. **`Application` entity contains 7 pre-aggregated/derived metrics** (totals, scores) — violates SLV-006/SLV-007.
9. **`Application_Customer` entity (62 attributes)** mixes party identity attributes with application-specific data — violates subject-area isolation (SLV-002 scope contamination from `silver.party`).
10. **`reference_application_ruleengine_cards*` (7 tables, up to 7.4 GB, 1.5M files)** — operational scoring engine data masquerading as Silver reference tables with no logical model backing.

### 1.2 Top 5 Immediate Actions

| Priority | Action | Effort | Owner |
|---|---|---|---|
| **P0-1** | Create all data mapping documents for all 49 physical tables; block Gold promotion until lineage exists | Large | ETL / Data Governance |
| **P0-2** | Remove `application_gold_fix`, `dummy_reference_education`, and `loan_gl_account` from `silver.application`; migrate `loan_gl_account` to `silver.gl` | Medium | DBA / Data Engineering |
| **P0-3** | Add `<entity>_sk` (MD5 surrogate) + `<entity>_bk` (business key) to every entity in the logical model and physical DDL | Large | Data Modeling |
| **P0-4** | Assign correct data types to all logical attributes (replace all STRING/CHAR(18) with proper TIMESTAMP, BOOLEAN, DECIMAL, INTEGER types) | Large | Data Modeling |
| **P0-5** | Add `Application_Workflow_Event` entity to capture onboarding TAT — this is a stated business requirement in `sa/subject-areas.md` that is entirely absent | Medium | Data Modeling / Business |

---

## 2. Subject Area Architecture Assessment

### 2.1 Scoping vs. `sa/subject-areas.md`

**Canonical definition (SA #16):**
> "Customer requests for products or services: applicant demographics, requested amounts, onboarding workflows, service requests, and TAT tracking."

**Assessment:** 🔴 **Non-compliant — over-scoped and under-scoped simultaneously**

**Over-scoped (entities/tables that do not belong here):**
- `loan_gl_account` — belongs in `silver.gl` (Finance GL SA)
- `reference_country`, `reference_currency` — belong in `silver.reference` (Reference Data SA)
- `reference_gender`, `reference_marital_status`, `reference_education`, `reference_occupation`, `reference_ethnicity` (implied) — belong in `silver.reference` or `silver.party`
- `reference_customer_type` — belongs in `silver.reference` or `silver.party`
- `reference_bank_branch` — belongs in `silver.org` (Organization SA)
- `application_gold_fix` — has no legitimate home in Silver
- `reference_application_ruleengine_cards*` (7 tables) — operational scoring engine data; belongs in Bronze or a dedicated Risk SA

**Under-scoped (required by SA definition but absent):**
- No `Application_Workflow_Event` entity for TAT tracking (explicitly required by SA description)
- No `Application_Document` entity for document submissions
- No `Application_Decision` entity for formal credit/service decisions
- No workflow state history table for tracking TAT milestones

**Confidence:** 0.95

### 2.2 BIAN / IFW Domain Alignment

**BIAN Mapping:** Application SA maps primarily to:
- **Product/Service Application** (BIAN Service Domain)
- **Customer Offer** and **Customer Agreement** (secondary)

**Assessment:** 🟡 **Partially aligned**

The `Application` entity correctly captures the application request lifecycle. However:
- `Application_Customer` duplicates attributes from BIAN's **Party** domain — applicant demographics should reference `silver.party.party` via a FK rather than being stored redundantly.
- `Application_Credit_Risk_Score` belongs more naturally in BIAN's **Credit Risk Assessment** domain (closer to `silver.risk`), but its presence here is acceptable given it is application-time scoring.
- The 7 `reference_application_ruleengine_cards*` tables appear to be raw decision-engine output from a CAPS/IBPS system — this is operational Bronze data, not a Silver reference entity.
- `application_loan` and `application_loan_ib` appear to be loan application detail tables — the existence of a separate IB (Islamic Banking) table suggests a structural split that should instead use a flag (`is_islamic_structure`) per the SA taxonomy's guidance on Islamic finance handling.

**Confidence:** 0.85

### 2.3 Identity Strategy

**Rule SLV-001:** `<entity>_sk` (MD5 surrogate) required on every entity.  
**Rule SLV-002:** `<entity>_bk` (business key) required on every entity.

**Assessment:** 🔴 **Entirely absent**

Not a single logical entity in the Application subject area defines a surrogate key (`_sk`) or a business key (`_bk`). The logical model shows no key attributes whatsoever — all attributes are payload columns. This means:
- The physical tables may have undocumented PKs not captured in the logical model
- SCD-2 history cannot function without deterministic surrogate keys
- Downstream Gold joins will be unreliable

**Evidence:** Logical model — all 28 entities lack any `_sk` or `_bk` attribute. Physical CSV has no column-level metadata to confirm or deny physical key existence.

**Confidence:** 0.90

### 2.4 SCD Strategy

**Rule SLV-003:** SCD-2 default; SCD-1 only with steward approval.

**Assessment:** 🔴 **Non-compliant**

The logical model defines SCD-like columns but with incorrect names and types:
- `Business Start Date` (CHAR(18)) → should be `effective_from` (TIMESTAMP)
- `Business End Date` (CHAR(18)) → should be `effective_to` (TIMESTAMP)
- `Is Active Flag` (CHAR(18)) → should be `is_current` (BOOLEAN)

Additionally:
- `run_id` is absent from all entities
- `source_ingestion_date` is absent from all entities
- Reference entities lack SCD columns entirely (e.g., `Reference_Customer_Type`, `Reference_Education`, `Reference_Country`)
- `Application_Feature` entity has no SCD columns at all

The use of CHAR(18) for date columns is a critical modelling error — CHAR(18) cannot store a TIMESTAMP value correctly.

**Confidence:** 0.92

### 2.5 Cross-Entity Referential Integrity

**Assessment:** 🔴 **No FKs defined**

The logical model defines no foreign key relationships between entities. Key missing FKs include:
- `Application` → `Application_Customer` (via application number + request type)
- `Application` → `Application_Credit_Risk_Score` (via application risk score id)
- `Application` → `Application_Feature` (via application id)
- `Application` → `Reference_Application_Status`
- `Application` → `Reference_Application_Type`
- `Application` → `Reference_Application_Source`
- `Application_Customer` → `Reference_Customer_Type`
- `Application_Customer` → `Reference_Gender`
- `Application_Customer` → `Reference_Education`
- `Application_Customer` → `Reference_Occupation`
- `Application_Customer` → `Reference_Marital_Status`
- `Application_Old_Employment` → `Application_Customer`
- `Application_Old_National_ID` → `Application_Customer`

Without FK definitions, the model cannot enforce referential integrity at Bronze→Silver gate or at Silver→Gold promotion.

**Confidence:** 0.90

### 2.6 ETL Mapping Completeness

**Assessment:** 🔴 **Zero mapping coverage**

No `*_data_mapping.xlsx` files were found in `sa/application/input/`. All 409 logical attribute rows show `mapping_present=None` and `mapping_avail=None`. This means:
- No lineage from source system to Silver column exists for any entity
- DQ rule derivation from business rules is impossible
- Gold promotion cannot be validated for correctness
- BCBS 239 lineage requirements are unmet for any attribute

This is the most critical single gap in the entire subject area.

**Confidence:** 1.0

---

## 3. Entity Inventory

| # | Logical Entity | Physical Table | Logical Attrs | Mapping Status | SK Present | BK Present | SCD-2 Cols | Issues |
|---|---|---|---|---|---|---|---|---|
| 1 | Application | `application` | 61 | 🔴 Unmapped | 🔴 No | 🔴 No | 🟡 Partial (wrong names/types) | Aggregations present; mixed concerns |
| 2 | Application_Customer | `application_customer` | 62 | 🔴 Unmapped | 🔴 No | 🔴 No | 🟡 Partial | Party data embedded; very wide; shadow `_new` table |
| 3 | Application_Credit_Risk_Score | `application_credit_risk_score` | 18 | 🔴 Unmapped | 🔴 No | 🔴 No | 🟡 Partial | Acceptable entity; missing keys |
| 4 | Application_Feature | `application_feature` | 10 | 🔴 Unmapped | 🔴 No | 🔴 No | 🔴 Missing | Empty table (0 files); no SCD cols |
| 5 | Application_Old_National_ID | **MISSING** | 15 | 🔴 Unmapped | 🔴 No | 🔴 No | 🔴 Missing | No physical table |
| 6 | Application_Old_Employment | `application_old_employment` | 15 | 🔴 Unmapped | 🔴 No | 🔴 No | 🟡 Partial | Acceptable entity; missing keys |
| 7 | Reference_Customer_Type | `reference_customer_type` | 8 | 🔴 Unmapped | 🔴 No | 🔴 No | 🔴 Missing | Should be in `silver.reference` |
| 8 | Reference_Country | `reference_country` | 16 | 🔴 Unmapped | 🔴 No | 🔴 No | 🔴 Missing | Should be in `silver.reference` |
| 9 | Reference_Ethnicity | **MISSING** | 9 | 🔴 Unmapped | 🔴 No | 🔴 No | 🔴 Missing | No physical table; scope dispute |
| 10 | Reference_Gender | `reference_gender` | 10 | 🔴 Unmapped | 🔴 No | 🔴 No | 🔴 Missing | Should be in `silver.reference` |
| 11 | Reference_ID_Type | `reference_id_type` | 9 | 🔴 Unmapped | 🔴 No | 🔴 No | 🔴 Missing | Acceptable scope (app-specific use) |
| 12 | Reference_Education | `reference_education` | 8 | 🔴 Unmapped | 🔴 No | 🔴 No | 🔴 Missing | Should be in `silver.reference` |
| 13 | Reference_Own_Home | `reference_own_home` | 8 | 🔴 Unmapped | 🔴 No | 🔴 No | 🔴 Missing | Empty table (0 files) |
| 14 | Reference_Marital_Status | `reference_marital_status` | 8 | 🔴 Unmapped | 🔴 No | 🔴 No | 🔴 Missing | Should be in `silver.reference` |
| 15 | Reference_Segment | **MISSING** | 9 | 🔴 Unmapped | 🔴 No | 🔴 No | 🔴 Missing | No physical table; likely in Party SA |
| 16 | Reference_Currency | `reference_currency` | 11 | 🔴 Unmapped | 🔴 No | 🔴 No | 🔴 Missing | Should be in `silver.reference` |
| 17 | Reference_Application_Status | `reference_application_status` | 10 | 🔴 Unmapped | 🔴 No | 🔴 No | 🔴 Missing | Correct scope |
| 18 | Reference_Application_Promotion | `reference_application_promotion` | 15 | 🔴 Unmapped | 🔴 No | 🔴 No | 🔴 Missing | Correct scope |
| 19 | Reference_Occupation | `reference_occupation` | 14 | 🔴 Unmapped | 🔴 No | 🔴 No | 🔴 Missing | Should be in `silver.reference` |
| 20 | Reference_Address | **STUB** (1 null attr) | 1 | 🔴 Unmapped | 🔴 No | 🔴 No | 🔴 Missing | Undefined entity; address_type physical table exists |
| 21 | Reference_Application_Source | `reference_application_source` | 9 | 🔴 Unmapped | 🔴 No | 🔴 No | 🔴 Missing | Correct scope |
| 22 | Reference_Application_Purpose | `reference_application_purpose` | 10 | 🔴 Unmapped | 🔴 No | 🔴 No | 🔴 Missing | Correct scope |
| 23 | Reference_Bank_Branch | `reference_bank_branch` | 18 | 🔴 Unmapped | 🔴 No | 🔴 No | 🔴 Missing | Should be in `silver.org` |
| 24 | Reference_Agency | `reference_agency` | 17 | 🔴 Unmapped | 🔴 No | 🔴 No | 🔴 Missing | Acceptable scope (app-specific) |
| 25 | Reference_Own_Car | `reference_own_car` | 8 | 🔴 Unmapped | 🔴 No | 🔴 No | 🔴 Missing | Empty table (0 files) |
| 26 | Reference_Facility | **MISSING** | 14 | 🔴 Unmapped | 🔴 No | 🔴 No | 🔴 Missing | No physical table |
| 27 | Reference_Application_Type | `reference_application_type` | 8 | 🔴 Unmapped | 🔴 No | 🔴 No | 🔴 Missing | Correct scope |
| 28 | Reference_Account_Status | `reference_account_status` | 8 | 🔴 Unmapped | 🔴 No | 🔴 No | 🔴 Missing | Should be in `silver.account` SA |

**Physical tables with NO logical entity (orphans):**

| Physical Table | Size | Assessment |
|---|---|---|
| `application_form` | 287 KB | No logical entity; likely raw form data |
| `application_gold_fix` | 285 MB | 🔴 **CRITICAL**: Gold workaround in Silver; must be removed |
| `application_loan` | 25 MB | No logical entity; loan application details — needs modelling |
| `application_loan_ib` | 25 MB | No logical entity; Islamic Banking loan application — needs modelling |
| `application_customer_new` | 17 MB | Shadow/replacement of `application_customer`; governance issue |
| `dummy_reference_education` | 8 KB | 🔴 **CRITICAL**: Development artifact in Silver |
| `loan_gl_account` | 20 KB | 🔴 Wrong SA — belongs in `silver.gl` |
| `reference_address_type` | 113 KB | No logical entity (Reference_Address stub) |
| `reference_application_loan` | 1.7 MB | No logical entity |
| `reference_application_ruleengine_cards` | 589 MB | No logical entity; raw rule engine data |
| `reference_application_ruleengine_cards_applicantdetails` | 331 MB | No logical entity; raw operational data |
| `reference_application_ruleengine_cards_capsbase` | **7.4 GB** | No logical entity; 1.54M files — operational store |
| `reference_application_ruleengine_cards_decision_history` | 0 bytes | No logical entity |
| `reference_application_ruleengine_cards_ibpsbase` | 0 bytes | No logical entity |
| `reference_application_ruleengine_cards_nationality` | 1.2 MB | No logical entity |
| `reference_application_ruleengine_cards_rejectionhistory` | 6.8 MB | No logical entity |
| `reference_application_ruleengine_cards_rejectionreasonsbase` | 0 bytes | No logical entity |
| `reference_application_ruleengine_cards_score_final` | 198 KB | No logical entity |
| `reference_channel_type` | 0 bytes | No logical entity |
| `reference_interest_index` | 0 bytes | No logical entity |
| `reference_loan_purpose` | 3 KB | No logical entity |
| `reference_residential_status` | 37 KB | No logical entity |
| `reference_score_value` | 2 MB | No logical entity |
| `reference_security_type_code` | 0 bytes | No logical entity |
| `reference_source_system` | 4.4 KB | No logical entity |

---

## 4. Per-Entity Assessments

---

### 4.1 Entity: Application

**Physical Table:** `silver.application.application`  
**Logical Attributes:** 61 | **Physical Files:** 2 (34.8 MB)

#### 4.1a Industry Fitness

🟡 **Partially correct**. The `Application` entity correctly captures the core application lifecycle (application number, dates, status, product details). However, it mixes multiple concerns:
- **Workflow management:** application status, status date, approved date (correct)
- **Product/facility details:** facility type, facility code, payment scheme (borderline — should reference Product SA)
- **Pre-computed aggregations:** Total loan amount, Total deposit balance, Total application score, Total fair market value — violates 3NF/SLV-007
- **Scoring denormalization:** `Application credit risk score` and `Total application Score` are embedded inline — the dedicated `Application_Credit_Risk_Score` entity already exists for this, creating duplication
- **Segment denormalization:** `Segment code` and `Segment description` are inline — description should not be stored (transitive dependency on reference table)

#### 4.1b Attribute-Level Review

| Attribute | Type in Model | Expected Type | Mapping | Issue |
|---|---|---|---|---|
| Application number | STRING | STRING NOT NULL | Unmapped | Should be `application_bk` (business key) |
| Application request type | STRING | STRING NOT NULL | Unmapped | Composite key component; no SK defined |
| Parent application number | STRING | STRING | Unmapped | FK to application_bk; not modelled |
| Application created date | STRING | DATE NOT NULL | Unmapped | Wrong type — should be DATE |
| Application status code | STRING | STRING NOT NULL | Unmapped | Should reference `reference_application_status`; raw code risks SLV-004 |
| Application status date | STRING | TIMESTAMP | Unmapped | Wrong type |
| Application promo code | STRING | STRING | Unmapped | FK to `reference_application_promotion` |
| Application purpose code | STRING | STRING | Unmapped | FK to `reference_application_purpose` |
| Facility number | STRING | STRING | Unmapped | FK to `Reference_Facility` (missing table) |
| Facility type | STRING | STRING | Unmapped | Should reference product catalog |
| Facility code | STRING | STRING | Unmapped | Should reference product catalog |
| Facility category id | STRING | STRING | Unmapped | Denormalized from product — should be FK |
| Application customer number | STRING | STRING NOT NULL | Unmapped | FK to `Application_Customer`; no FK defined |
| Core banking customer account number | STRING | STRING | Unmapped | Cross-SA reference to `silver.account`; not modelled |
| Application branch code | STRING | STRING | Unmapped | FK to `reference_bank_branch` (wrong SA) |
| Agency name | STRING | STRING | Unmapped | Denormalized description; FK to `reference_agency` |
| Proposed value | STRING | DECIMAL(18,4) | Unmapped | Wrong type; amount without currency suffix |
| Product category | STRING | STRING | Unmapped | Denormalized from product catalog |
| Application process type | STRING | STRING | Unmapped | No reference table defined |
| Personal use indicator | STRING | BOOLEAN | Unmapped | Wrong type — should be `is_personal_use` BOOLEAN |
| Payment scheme | STRING | STRING | Unmapped | Raw source code risk (SLV-004) |
| Monthly installment amount | STRING | DECIMAL(18,4) | Unmapped | 🔴 Wrong type; missing currency suffix (`_amount_aed`) |
| Loan tenure | STRING | INTEGER | Unmapped | Wrong type |
| Loan limit amount | STRING | DECIMAL(18,4) | Unmapped | 🔴 Wrong type; missing currency suffix |
| Installment scheme | STRING | STRING | Unmapped | No reference table |
| Existing loan outstanding amount | STRING | DECIMAL(18,4) | Unmapped | 🔴 Wrong type; missing currency suffix |
| Effective interest rate | STRING | DECIMAL(10,6) | Unmapped | Wrong type; rate should not be STRING |
| Approved date | STRING | DATE | Unmapped | Wrong type |
| Negative hit indicator | STRING | BOOLEAN | Unmapped | Wrong type — should be `is_negative_hit` BOOLEAN |
| Application credit risk score | STRING | DECIMAL(10,4) | Unmapped | 🔴 Duplicates `Application_Credit_Risk_Score` entity |
| Address type | STRING | STRING | Unmapped | FK reference undefined |
| Undisbursed amount | STRING | DECIMAL(18,4) | Unmapped | 🔴 Wrong type; missing currency suffix |
| **Total loan amount** | STRING | N/A | Unmapped | 🔴 **AGGREGATION — violates SLV-007** |
| **Total gross monthly income amount** | STRING | N/A | Unmapped | 🔴 **AGGREGATION — violates SLV-007** |
| Total family size | STRING | INTEGER | Unmapped | Wrong type |
| **Total deposit balance amount** | STRING | N/A | Unmapped | 🔴 **AGGREGATION — violates SLV-007** |
| **Total application Score** | STRING | N/A | Unmapped | 🔴 **DERIVED METRIC — violates SLV-007** |
| **Total acceptable monthly income amount final grade** | STRING | N/A | Unmapped | 🔴 **DERIVED METRIC — violates SLV-007** |
| **Total limit amount** | STRING | N/A | Unmapped | 🔴 **AGGREGATION — violates SLV-007** |
| **Total fair market value collateral amount** | STRING | N/A | Unmapped | 🔴 **AGGREGATION/CROSS-SA — violates SLV-007** |
| Segment code | STRING | STRING | Unmapped | Denormalized; FK to reference_segment (no table) |
| Segment description | STRING | STRING | Unmapped | 🔴 Transitive dependency — description from code |
| Scoring customer cif key | STRING | STRING | Unmapped | Possible duplicate of Application customer number |
| Approved total acceptable monthly income | STRING | DECIMAL(18,4) | Unmapped | Wrong type; aggregation concern |
| Application score indicator | STRING | STRING | Unmapped | Scoring denormalization |
| Cash downpayment amount | STRING | DECIMAL(18,4) | Unmapped | 🔴 Wrong type; missing currency suffix |
| Current bank financier | STRING | STRING | Unmapped | Free-text field; should be FK to reference |
| Source system code | STRING | STRING NOT NULL | Unmapped | Present ✅ but wrong case (not snake_case `source_system_code`) |
| Source system id | STRING | STRING | Unmapped | Present ✅ |
| IsLogicallyDeleted | STRING | BOOLEAN | Unmapped | 🔴 Wrong type AND wrong name (not snake_case) |
| Create date | STRING | TIMESTAMP NOT NULL | Unmapped | 🔴 Wrong type (STRING) |
| Update date | STRING | TIMESTAMP | Unmapped | 🔴 Wrong type (STRING) |
| Deleted date | STRING | TIMESTAMP | Unmapped | 🔴 Wrong type (STRING) |
| Application type code | STRING | STRING NOT NULL | Unmapped | FK to `reference_application_type` |
| Application source code | STRING | STRING | Unmapped | FK to `reference_application_source` |
| Application status reason type code | STRING | STRING | Unmapped | FK to `reference_application_status` |
| Application system customer number | STRING | STRING | Unmapped | Relationship to Application_Customer |
| Application risk score id | STRING | STRING | Unmapped | FK to `Application_Credit_Risk_Score` |
| Business Start Date | CHAR(18) | TIMESTAMP NOT NULL | Unmapped | 🔴 Wrong type AND wrong name — should be `effective_from TIMESTAMP` |
| Business End Date | CHAR(18) | TIMESTAMP | Unmapped | 🔴 Wrong type AND wrong name — should be `effective_to TIMESTAMP` |
| Is Active Flag | CHAR(18) | BOOLEAN NOT NULL | Unmapped | 🔴 Wrong type AND wrong name — should be `is_current BOOLEAN` |

**Missing from logical model (required by guideline):**
- `application_sk` (STRING NOT NULL) — MD5 surrogate key
- `application_bk` (STRING NOT NULL) — business key
- `run_id` (STRING) — pipeline run identifier
- `source_ingestion_date` (TIMESTAMP) — source emission timestamp
- `_dq_status` / `data_quality_status` (STRING) — DQ outcome
- `_dq_flags` / `dq_issues` (STRING) — DQ rule failure details

#### 4.1c Metadata Completeness

| Required Column | Present in Model | Type in Model | Expected Type | Compliant |
|---|---|---|---|---|
| `source_system_code` | ✅ (as "Source system code") | STRING | STRING | 🟡 Name casing |
| `source_system_id` | ✅ (as "Source system id") | STRING | STRING | 🟡 Name casing |
| `create_date` | ✅ (as "Create date") | STRING | TIMESTAMP | 🔴 Wrong type |
| `update_date` | ✅ (as "Update date") | STRING | TIMESTAMP | 🔴 Wrong type |
| `delete_date` | ✅ (as "Deleted date") | STRING | TIMESTAMP | 🔴 Wrong type |
| `is_active_flag` | ✅ (as "Is Active Flag") | CHAR(18) | STRING | 🔴 Wrong type + PascalCase name |
| `effective_from` | ✅ (as "Business Start Date") | CHAR(18) | TIMESTAMP NOT NULL | 🔴 Wrong type + wrong name |
| `effective_to` | ✅ (as "Business End Date") | CHAR(18) | TIMESTAMP | 🔴 Wrong type + wrong name |
| `is_current` | ✅ (as "Is Active Flag" approximate) | CHAR(18) | BOOLEAN NOT NULL | 🔴 Wrong type + wrong name |
| `run_id` | 🔴 Missing | — | STRING | 🔴 Absent |
| `source_ingestion_date` | 🔴 Missing | — | TIMESTAMP | 🔴 Absent |
| `data_quality_status` | 🔴 Missing | — | STRING | 🔴 Absent |
| `dq_issues` | 🔴 Missing | — | STRING | 🔴 Absent |

---

### Finding Application-001 — No Surrogate Key or Business Key Defined

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | `SLV-001` — "Every Silver entity must have a surrogate key `<entity>_sk` derived as `MD5(UPPER(TRIM(business_key)))`"; `SLV-002` — "Every Silver entity must have `<entity>_bk` as the natural/business key" |
| Evidence | Logical model: Application entity (61 attrs) — no `application_sk` or `application_bk` attribute present |
| Affected Table | `silver.application.application` |
| Affected Column(s) | `application_sk` (missing), `application_bk` (missing) |
| Confidence | 1.0 |

**Description:** The `Application` entity — and every other entity in this subject area — lacks both a surrogate key (`application_sk`) and a business key (`application_bk`). The application number + request type together form the natural composite key, but neither has been formalised. Without a surrogate key, SCD-2 history is non-functional and Gold FK joins are unreliable.

**Remediation:** Add `application_sk STRING NOT NULL` (MD5 of UPPER(TRIM(application_number || '|' || application_request_type))) and `application_bk STRING NOT NULL` to the logical model and physical DDL. Apply across all 28 entities.

**Estimated Effort:** Large  
**Owner:** Data Modeling

---

### Finding Application-002 — Seven Pre-Aggregated/Derived Metrics Violate SLV-007

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | `SLV-006` — "3NF; no derived values or aggregations in Silver"; `SLV-007` — "No pre-computed metrics in Silver" |
| Evidence | Logical model: Application entity attributes — "Total loan amount", "Total gross monthly income amount", "Total deposit balance amount", "Total application Score", "Total acceptable monthly income amount final grade", "Total limit amount", "Total fair market value collateral amount" |
| Affected Table | `silver.application.application` |
| Affected Column(s) | `total_loan_amount`, `total_gross_monthly_income_amount`, `total_deposit_balance_amount`, `total_application_score`, `total_acceptable_monthly_income_amount_final_grade`, `total_limit_amount`, `total_fair_market_value_collateral_amount` |
| Confidence | 0.95 |

**Description:** Seven attributes in the `Application` entity are pre-aggregated totals or derived scores. These violate SLV-006/SLV-007 — Silver must store only atomic, un-aggregated facts. Furthermore, `total_fair_market_value_collateral_amount` crosses into the Collateral SA, constituting a subject-area isolation violation. `Total deposit balance amount` crosses into the Account SA.

**Remediation:** Remove all seven aggregation/derived attributes from the `Application` entity. Atomic applicant income should be in `Application_Customer.gross_monthly_income_amount`. Collateral fair market value belongs in `silver.collateral`. Totals should be computed at Gold layer. If application-time snapshots are required for regulatory or audit purposes, create a separate `application_snapshot` entity with explicit steward approval.

**Estimated Effort:** Medium  
**Owner:** Data Modeling / Business

---

### Finding Application-003 — All Attribute Types Defined as STRING or CHAR(18)

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | `SLV-010` — "Monetary amounts in smallest unit (fils for AED)"; guideline §2 DQ standard — "All monetary columns must use DECIMAL(18,4)"; modeling rule 2.7 — explicit types required |
| Evidence | Logical model: All 61 Application attributes typed as STRING or CHAR(18), including date fields, boolean indicators, monetary amounts, integer counts, and percentage/rate fields |
| Affected Table | `silver.application.application` |
| Affected Column(s) | All attributes — see attribute table above |
| Confidence | 1.0 |

**Description:** The entire logical model for Application SA uses STRING for every attribute regardless of semantic type. This means:
- Date columns (application_created_date, approved_date) are STRING — no date validation possible
- Monetary amounts (monthly_installment_amount, loan_limit_amount, etc.) are STRING — arithmetic impossible; precision/rounding undefined
- Boolean indicators (personal_use_indicator, negative_hit_indicator, IsLogicallyDeleted) are STRING — type-unsafe comparisons
- SCD columns (Business Start/End Date) are CHAR(18) — cannot represent a TIMESTAMP
- Count fields (total_family_size, number_of_dependents) are STRING — summation will fail

**Remediation:** Assign correct SQL types to every attribute. Monetary amounts → DECIMAL(18,4) with `_amount_aed` or `_amount_orig` suffix. Date columns → DATE or TIMESTAMP as appropriate. Boolean indicators → BOOLEAN. Counts/integers → INT. Rates/percentages → DECIMAL(10,6).

**Estimated Effort:** Large  
**Owner:** Data Modeling

---

### Finding Application-004 — SCD-2 Columns Incorrectly Named and Typed

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | `SLV-003` — "SCD-2 default; SCD-1 only with steward approval"; modeling rule 2.7 — `effective_from TIMESTAMP`, `effective_to TIMESTAMP`, `is_current BOOLEAN`, `run_id STRING`, `source_ingestion_date TIMESTAMP` |
| Evidence | Logical model: "Business Start Date" (CHAR(18)), "Business End Date" (CHAR(18)), "Is Active Flag" (CHAR(18)) |
| Affected Table | `silver.application.application` |
| Affected Column(s) | `Business Start Date`, `Business End Date`, `Is Active Flag` |
| Confidence | 1.0 |

**Description:** The SCD-2 tracking columns use non-standard names and incorrect types. "Business Start Date" / "Business End Date" do not match the required `effective_from` / `effective_to` names. CHAR(18) cannot store a TIMESTAMP value. `Is Active Flag` should be `is_current` with type BOOLEAN. Additionally, `run_id` and `source_ingestion_date` are absent entirely. The same pattern applies to all 6 core application entities.

**Remediation:** Rename to `effective_from TIMESTAMP NOT NULL`, `effective_to TIMESTAMP`, `is_current BOOLEAN NOT NULL`. Add `run_id STRING` and `source_ingestion_date TIMESTAMP`. Update physical DDL to match. Apply to all 28 logical entities.

**Estimated Effort:** Large  
**Owner:** Data Modeling / ETL

---

### Finding Application-005 — No Data Mapping (Zero ETL Lineage Coverage)

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | `SLV-005` — "DQ-gated writes; quarantine table"; BCBS 239 Principle 2 — "Data Aggregation Capabilities" requires documented lineage |
| Evidence | `sa/application/input/` directory contains only `application_physical_structures.csv` — no `*_data_mapping.xlsx` files; all 409 logical attribute rows have `mapping_present=None` and `mapping_avail=None` |
| Affected Table | All 49 `silver.application.*` tables |
| Affected Column(s) | All columns |
| Confidence | 1.0 |

**Description:** Not a single column in the Application subject area has a documented source-to-target mapping. This means ETL pipelines have been built (49 physical tables exist with data) without any formalized mapping documentation. DQ rules cannot be defined, lineage cannot be traced, Gold promotion cannot be validated, and BCBS 239 lineage requirements are entirely unmet. The `application` table has 34.8 MB of actual data with no mapping documentation.

**Remediation:** Create source-to-target mapping workbooks for all 49 physical tables. Prioritize: `application`, `application_customer`, `application_loan`, `application_loan_ib`, `application_credit_risk_score`. Each mapping must include: source system, source table, source column, transformation rule, target column, DQ rule applied.

**Estimated Effort:** Large  
**Owner:** ETL / Data Governance

---

### 4.2 Entity: Application_Customer

**Physical Table:** `silver.application.application_customer`  
**Logical Attributes:** 62 | **Physical Files:** 28 (393 KB)  
**Shadow Table:** `application_customer_new` (17.9 MB — no logical entity)

#### 4.2a Industry Fitness

🔴 **Problematic — violates subject-area isolation**. The `Application_Customer` entity (62 attributes) co-locates:
1. **Application-specific applicant facts** (relation to main applicant, income document type, employment type, number of credit cards for this application) — correct scope
2. **Party/KYC attributes** (nationality, gender, education, occupation, marital status, age, birth date, country of incorporation) — these belong in `silver.party.party`, not duplicated here
3. **Credit bureau data** (credit bureau score, asset liability ratio, bank balance) — borderline; acceptable as application-time snapshot
4. **Aggregated/derived fields** (asset liability ratio) — SLV-007 concern

The entity is significantly over-wide (62 attributes) and will create maintenance burden and data drift as Party SA attributes evolve.

**Recommendation:** Refactor to keep only application-specific attributes. Party demographic attributes should be replaced with a FK to `silver.party.party_sk`. If application-time snapshots of party attributes are required for regulatory purposes, create `Application_Customer_Party_Snapshot` as an explicit snapshot entity.

#### 4.2b Selected Attribute Issues

| Attribute | Type in Model | Expected Type | Issue |
|---|---|---|---|
| Application number | STRING | STRING NOT NULL | Should be composite FK to Application |
| Application System Customer number | STRING | STRING NOT NULL | Should be `application_customer_bk` |
| Age | STRING | INTEGER | 🔴 Wrong type; age is derived from birth date — possible SLV-007 violation |
| Birth or Incorporation date | STRING | DATE | 🔴 Wrong type |
| Race | STRING | STRING | Sensitive PII — requires masking per DQ standard §2 Gate 2 |
| National Identifier or Registry of Company number | STRING | STRING | 🔴 PII — must be SHA-256 masked per DQ standard |
| Old national id number | STRING | STRING | 🔴 PII — must be SHA-256 masked |
| Credit bureau score | STRING | DECIMAL(10,4) | 🔴 Wrong type |
| Asset liability ratio | STRING | DECIMAL(10,6) | 🔴 Wrong type; derived ratio — SLV-007 concern |
| Bank balance amount | STRING | DECIMAL(18,4) | 🔴 Wrong type; cross-SA (Account) concern |
| Gross monthly income amount | STRING | DECIMAL(18,4) | 🔴 Wrong type; missing currency suffix |
| Acceptable monthly income amount | STRING | DECIMAL(18,4) | 🔴 Wrong type; derived/adjusted figure |
| Number of dependents | STRING | INTEGER | 🔴 Wrong type |
| Number of credit cards | STRING | INTEGER | 🔴 Wrong type |
| Self-employed indicator | STRING | BOOLEAN | 🔴 Wrong type |
| Negative hit indicator | STRING | BOOLEAN | 🔴 Wrong type |
| My info indicator | STRING | BOOLEAN | 🔴 Wrong type |
| Present address duration | STRING | INTEGER | 🔴 Wrong type (months/years) |
| Business Start Date | CHAR(18) | TIMESTAMP NOT NULL | 🔴 Wrong type + wrong name |
| Business End Date | CHAR(18) | TIMESTAMP | 🔴 Wrong type + wrong name |
| Is Active Flag | CHAR(18) | BOOLEAN NOT NULL | 🔴 Wrong type + wrong name |
| IsLogicallyDeleted | STRING | BOOLEAN | 🔴 Wrong type + not snake_case |

#### 4.2c Metadata Completeness

Same gaps as `Application` entity: `run_id` missing, `source_ingestion_date` missing, DQ columns missing, SCD columns incorrectly named/typed.

**Shadow Table Issue:** `application_customer_new` (17.9 MB, no logical entity) appears to be a structural replacement or parallel load of `application_customer`. This creates ambiguity about which table is authoritative. No catalog description exists (null). This must be resolved with either deprecation or promotion to the canonical table.

---

### Finding Application-006 — Party Attributes Embedded in Application_Customer

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | High |
| Guideline Rule | Modeling rule 2.2 — "Keep each Silver subject area self-contained... do not join across subject areas inside Silver pipelines"; SLV-006 — "3NF; no derived values or aggregations" |
| Evidence | Logical model: Application_Customer entity contains nationality, gender, education, marital_status, occupation, age, birth_date, country attributes — identical to Party SA attributes |
| Affected Table | `silver.application.application_customer` |
| Affected Column(s) | `Race`, `Nationality`, `Gender`, `Education code`, `Occupation code`, `Marital status`, `Age`, `Birth or Incorporation date`, `Country of incorporation`, `Customer type`, `Customer sub type`, `Country of domicile`, `Constitution` |
| Confidence | 0.90 |

**Description:** `Application_Customer` duplicates Party attributes that are maintained in `silver.party`. This creates data drift: when a customer's nationality or education changes in Party SA, the Application SA copy becomes stale. For regulatory reporting, the Party SA is the system of record for these attributes. The application-time snapshot argument is valid only for certain attributes (income, employment status at application time), not for static identity attributes like nationality or birth date.

**Remediation:** Remove static party identity attributes from `Application_Customer`. Add `party_sk STRING` as a FK to `silver.party.party`. Retain only application-specific applicant attributes: relation to main applicant, income document type, employment type at application time, number of dependents declared on the application.

**Estimated Effort:** Large  
**Owner:** Data Modeling / Business (requires sign-off on which attributes are application-time vs. party-canonical)

---

### Finding Application-007 — Shadow Table `application_customer_new` Has No Logical Entity

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | Modeling rule 2.11 — "Create and approve the logical and physical model in Erwin before generating DDL. Tables that exist without a model entry are a governance gap" |
| Evidence | Physical: `silver.application.application_customer_new` (17.9 MB, 1 file); Logical: no corresponding entity in model |
| Affected Table | `silver.application.application_customer_new` |
| Affected Column(s) | All |
| Confidence | 1.0 |

**Description:** A physical table `application_customer_new` exists with 17.9 MB of data but has no corresponding logical model entity, no description, and no mapping documentation. The "_new" suffix suggests it was created as a replacement during a migration or schema change, but was never formalized or deprecated. This constitutes a governance gap. It may contain authoritative data that consumers are using without awareness.

**Remediation:** (1) Determine if this is the intended production table replacing `application_customer`. (2) If yes — formalize in the logical model, deprecate `application_customer` per the versioning procedure (6-month retention), update all pipelines. (3) If no — drop it after confirming no downstream consumers.

**Estimated Effort:** Small  
**Owner:** Data Engineering / Data Governance

---

### Finding Application-008 — PII Attributes Require SHA-256 Masking

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | DQ guideline §2 Gate 2 — "EID, Card Numbers, Phone Numbers, and Passport Numbers masked via SHA-256 unless stored in a restricted container"; DQ guideline §10 — PDPL Article 5 consent requirements |
| Evidence | Logical model: Application_Customer contains "National Identifier or Registry of Company number", "Old national id number", "Race", "Telephone type" |
| Affected Table | `silver.application.application_customer` |
| Affected Column(s) | `national_identifier_or_registry_of_company_number`, `old_national_id_number` |
| Confidence | 0.95 |

**Description:** National ID numbers and Emirates ID numbers are classified PII under PDPL and must be SHA-256 masked in Silver (unless in a restricted container with role-based access). The logical model shows no masking annotation or PII classification on these attributes. `Race` is sensitive demographic data requiring explicit classification and restricted access.

**Remediation:** Tag `national_identifier` and `old_national_id` with PII classification in the data catalog. Apply SHA-256 masking in the Bronze→Silver ETL pipeline. Ensure `Race` attribute has an access-control annotation in the catalog with restricted analyst roles. Document retention and erasure procedures per PDPL Article 5.

**Estimated Effort:** Medium  
**Owner:** Data Governance / ETL

---

### 4.3 Entity: Application_Credit_Risk_Score

**Physical Table:** `silver.application.application_credit_risk_score`  
**Logical Attributes:** 18 | **Physical Files:** 3 (184 KB)

#### 4.3a Industry Fitness

🟡 **Reasonable scope** — captures application-time credit risk scoring, which is appropriately an application-level event. However, a naming concern exists: this should be `application_risk_score` as "credit risk" pre-empts the risk domain owned by the Risk & Compliance SA.

**Key concern:** The parent `Application` entity also has inline attributes `Application credit risk score` and `Application score indicator` — these duplicate the scoring entity and should be removed from `Application` in favour of the FK to this entity.

#### 4.3b Selected Attribute Issues

| Attribute | Type in Model | Expected Type | Issue |
|---|---|---|---|
| Application number | STRING | STRING NOT NULL | Composite FK — should be `application_sk` FK |
| Application request type | STRING | STRING NOT NULL | Composite FK component |
| Application system customer number | STRING | STRING | FK to Application_Customer |
| Application risk score id | STRING | STRING NOT NULL | Should be `application_credit_risk_score_bk` |
| Risk score | STRING | DECIMAL(10,4) | 🔴 Wrong type |
| Risk grade | STRING | STRING | FK to reference table (missing) |
| Risk category | STRING | STRING | FK to reference table (missing) |
| Score model | STRING | STRING | FK to reference table (missing) |
| Score date | STRING | DATE | 🔴 Wrong type |
| Business Start Date | CHAR(18) | TIMESTAMP NOT NULL | 🔴 Wrong type + wrong name |
| Business End Date | CHAR(18) | TIMESTAMP | 🔴 Wrong type + wrong name |
| Is Active Flag | CHAR(18) | BOOLEAN NOT NULL | 🔴 Wrong type + wrong name |

---

### Finding Application-009 — Risk Score Duplicated Between Application and Application_Credit_Risk_Score

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | Medium |
| Guideline Rule | SLV-006 — "3NF; no derived values/aggregations"; modeling rule 2.1 — no transitive dependencies |
| Evidence | Logical model: Application entity has "Application credit risk score" and "Application score indicator"; Application_Credit_Risk_Score entity has "Risk score", "Risk grade", "Risk category", "Score model" |
| Affected Table | `silver.application.application`, `silver.application.application_credit_risk_score` |
| Affected Column(s) | `application.application_credit_risk_score`, `application.total_application_score`, `application.application_score_indicator` |
| Confidence | 0.90 |

**Description:** Credit score data is stored in two places: inline in the `Application` entity and in the dedicated `Application_Credit_Risk_Score` entity. This is a 3NF violation. The canonical location is the scoring entity.

**Remediation:** Remove `application_credit_risk_score`, `total_application_score`, and `application_score_indicator` from the `Application` entity. Store only the FK `application_risk_score_id` referencing `Application_Credit_Risk_Score`.

**Estimated Effort:** Small  
**Owner:** Data Modeling

---

### 4.4 Entity: Application_Feature

**Physical Table:** `silver.application.application_feature`  
**Logical Attributes:** 10 | **Physical Files:** 0 (0 bytes — empty table)

#### 4.4a Industry Fitness

🟡 **Appropriate scope** — captures product features negotiated at application time (interest rate, amount, tenure). The concept maps to BIAN's "Product Feature" applied to an application.

#### 4.4b Attribute Issues

**Note:** Attribute names below are reproduced verbatim from the logical model to document naming violations. Recommended canonical names are given in the Issue column.

| Attribute (current model name — non-compliant) | Type in Model | Expected Type | Issue |
|---|---|---|---|
| `Feature_Id` | STRING | STRING NOT NULL | 🔴 PascalCase violates NAM-001; rename to `application_feature_bk` (business key) or `feature_id`; no SK defined |
| `Application Feature Start Date` | STRING | DATE | 🔴 Spaces in name violate NAM-001; rename to `application_feature_start_date DATE` |
| `Application Id` | STRING | STRING NOT NULL | 🔴 Spaces violate NAM-001; rename to `application_id` or `application_bk`; FK undefined |
| `Application Overridden Feature ID` | STRING | STRING | 🔴 Spaces + mixed case violate NAM-001; rename to `application_overridden_feature_id` |
| `Application Feature End Date` | STRING | DATE | 🔴 Spaces violate NAM-001; rename to `application_feature_end_date DATE` |
| `Application Feature UOM Cd` | STRING | STRING | 🔴 Non-standard abbreviation `Cd` and spaces violate NAM-001; rename to `unit_of_measure_code` |
| `Application Feature Amount` | STRING | DECIMAL(18,4) | 🔴 Spaces violate NAM-001; rename to `application_feature_amount_aed`; wrong type |
| `Application Feature Rate` | STRING | DECIMAL(10,6) | 🔴 Spaces violate NAM-001; rename to `application_feature_rate`; wrong type |
| `Application Feature Quantity` | STRING | INTEGER | 🔴 Spaces violate NAM-001; rename to `application_feature_quantity`; wrong type |
| `Application Feature Number` | STRING | INTEGER | 🔴 Spaces violate NAM-001; rename to `application_feature_number`; wrong type; unclear semantics |

**Critical gap:** `Application_Feature` has **no SCD-2 columns** whatsoever (no effective_from, effective_to, is_current, create_date, update_date). This entity cannot track feature changes over time.

---

### Finding Application-010 — Application_Feature Missing All Metadata/SCD Columns

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | Modeling rule 2.7 — "Include the following Silver technical audit columns on every Silver entity table"; SLV-003 — "SCD-2 default" |
| Evidence | Logical model: Application_Feature (10 attrs) — no `source_system_code`, `source_system_id`, `create_date`, `update_date`, `delete_date`, `is_active_flag`, `effective_from`, `effective_to`, `is_current`, `run_id`, `source_ingestion_date` |
| Affected Table | `silver.application.application_feature` |
| Affected Column(s) | All metadata columns — missing entirely |
| Confidence | 1.0 |

**Description:** The `Application_Feature` entity is entirely missing the 11 mandatory Silver technical audit columns. The physical table also has 0 files (empty), suggesting the entity was scaffolded but never implemented. Without metadata columns, DQ tracking, SCD history, and lineage are impossible.

**Remediation:** Add all 11 required audit columns to the logical model and physical DDL. Implement the ETL pipeline to populate this entity.

**Estimated Effort:** Medium  
**Owner:** Data Modeling / ETL

---

### 4.5 Entity: Application_Old_National_ID

**Physical Table:** **MISSING** — no `application_old_national_id` table in physical structures  
**Logical Attributes:** 15

#### 4.5a Assessment

🔴 **Unmapped — no physical table**. The entity is defined in the logical model but has no physical implementation. Given that `Application_Old_Employment` (a parallel entity) has a physical table, the missing `Application_Old_National_ID` table is a gap.

#### 4.5b Attribute Issues (Logical Only)

The entity contains `Old National id number` and `Old national Id expiration date` — the ID number is PII and requires SHA-256 masking. Type issues mirror other entities: all STRING/CHAR(18). Business Start/End Date with CHAR(18) type are present.

---

### Finding Application-011 — Application_Old_National_ID Has No Physical Table

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | Medium |
| Guideline Rule | Modeling rule 2.11 — "Tables must have a logical model entry; entities in the model must have physical tables" |
| Evidence | Logical model: Application_Old_National_ID (15 attrs); Physical: no matching table in 49-table list |
| Affected Table | `silver.application.application_old_national_id` (required, missing) |
| Affected Column(s) | All |
| Confidence | 1.0 |

**Description:** The `Application_Old_National_ID` entity exists in the logical model but has no corresponding physical Delta table. Historical national ID data for applicants cannot be stored or queried. This may be important for identity verification and KYC audit trails.

**Remediation:** Create `silver.application.application_old_national_id` DDL from the logical model after correcting attribute types and adding SK/BK and audit columns. Implement ETL pipeline. Apply PII masking to `old_national_id_number`.

**Estimated Effort:** Medium  
**Owner:** Data Engineering / Data Modeling

---

### 4.6 Entity: Application_Old_Employment

**Physical Table:** `silver.application.application_old_employment`  
**Logical Attributes:** 15 | **Physical Files:** 1 (90 KB)

#### 4.6a Industry Fitness

🟢 **Appropriate scope** — prior employment history captured at application time. Correct as a child entity of `Application_Customer`.

#### 4.6b Key Attribute Issues

- `Application number` + `Application request type` form the composite FK to parent `Application` — no FK constraint defined
- `Old employer number` is STRING but should be a FK reference — no reference entity for employers defined
- `Employment end date` / `Employment start date` are STRING — should be DATE
- All standard type/naming issues apply (SCD columns incorrectly named/typed)

---

### 4.7 Reference Entities (Batch Assessment)

All 22 reference entities share the following common gaps:

**Universal Gaps:**
1. **No surrogate key or business key** on any reference entity
2. **No SCD-2 columns** (most lack effective_from/to entirely; a few have the incorrect "Business Start/End Date" CHAR(18))
3. **All attribute types are STRING** — code columns should be STRING, but description/date/boolean columns need correct types
4. **No DQ columns** (`data_quality_status`, `dq_issues`)
5. **`run_id` absent** from all
6. **`source_ingestion_date` absent** from all
7. **`IsLogicallyDeleted` as STRING** — should be `is_logically_deleted BOOLEAN` or use `is_active_flag`

---

### Finding Application-012 — Reference Tables That Belong in `silver.reference`

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | High |
| Guideline Rule | Modeling rule 2.2 — "Keep each Silver subject area self-contained"; `sa/subject-areas.md` SA #1 Reference Data — "Master Lookups: ISO Currencies, Country Codes, ... Industry Codes (NACE/SIC)" |
| Evidence | Physical: `reference_country`, `reference_currency`, `reference_gender`, `reference_education`, `reference_marital_status`, `reference_occupation`, `reference_customer_type` in `silver.application`; SA taxonomy: these belong in `silver.reference` |
| Affected Table | 7 tables listed above |
| Affected Column(s) | Entire tables |
| Confidence | 0.95 |

**Description:** Seven reference tables contain domain-wide master data (country codes, currencies, gender codes, education levels, occupation types, marital status codes, customer types) that are logically part of the canonical Reference Data subject area (`silver.reference`). Having duplicated copies in `silver.application` means: (1) code mappings can diverge between SAs, (2) SLV-004 code canonicalization is violated since each SA maintains its own local reference copy, (3) maintenance overhead is doubled.

**Remediation:** Migrate these tables to `silver.reference`. Application SA pipelines should JOIN to `silver.reference.*` for code lookups rather than maintaining local copies. If application-specific overrides are needed, use the `silver.reference.code_mapping` pattern.

**Estimated Effort:** Medium  
**Owner:** Data Architecture / Data Engineering

---

### Finding Application-013 — `Reference_Bank_Branch` Belongs in Organization SA

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | Medium |
| Guideline Rule | `sa/subject-areas.md` SA #18 Organization — "Internal Bank Structure: Branches, Departments, Cost Centers, Regions, ATMs"; modeling rule 2.2 — subject-area isolation |
| Evidence | Logical model: `Reference_Bank_Branch` (18 attrs) in Application SA; Physical: `reference_bank_branch` in `silver.application`; SA taxonomy: branches belong in `silver.org` |
| Affected Table | `silver.application.reference_bank_branch` |
| Affected Column(s) | Entire table |
| Confidence | 0.95 |

**Description:** Branch master data is canonical Organization SA data. Duplicating it in Application SA creates version drift — when branches open/close/merge, the Application SA copy may not be updated. Branch attributes (IFSC code, routing number, merge date) are organization data, not application data.

**Remediation:** Remove `Reference_Bank_Branch` from Application SA. The Application entity should reference branch via `branch_code` FK to `silver.org.branch`.

**Estimated Effort:** Small  
**Owner:** Data Architecture

---

### Finding Application-014 — `Reference_Account_Status` Belongs in Account SA

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | Medium |
| Guideline Rule | `sa/subject-areas.md` SA #5 Account — "Holds Lifecycle status"; modeling rule 2.2 |
| Evidence | Logical model: `Reference_Account_Status` (8 attrs) in Application SA; SA taxonomy: account status belongs in `silver.account` |
| Affected Table | `silver.application.reference_account_status` |
| Affected Column(s) | Entire table |
| Confidence | 0.85 |

**Description:** Account status codes are Account SA data. An application may _result in_ an account, but the account status reference table belongs in `silver.account`. Having it here creates a second authoritative source for account status codes.

**Remediation:** Remove from Application SA. Reference `silver.account.account_status` for code lookups.

**Estimated Effort:** Small  
**Owner:** Data Architecture

---

### Finding Application-015 — `Reference_Ethnicity` Has No Physical Table

| Field | Value |
|---|---|
| Priority | P2 |
| Criticality | Low |
| Guideline Rule | Modeling rule 2.11 — entities in model must have physical tables |
| Evidence | Logical model: `Reference_Ethnicity` (9 attrs); Physical: no `reference_ethnicity` table in 49-table list |
| Affected Table | `silver.application.reference_ethnicity` (required, missing) |
| Affected Column(s) | All |
| Confidence | 1.0 |

**Description:** `Reference_Ethnicity` exists in the logical model but has no physical implementation. Note: ethnicity data has PII/sensitive classification implications — storage of ethnicity codes requires explicit legal basis under UAE PDPL.

**Remediation:** Either (1) create the physical table after legal review confirms a processing basis for ethnicity data, or (2) remove the entity from the logical model if no use case requires it. Given sensitivity, escalate to Data Governance before implementation.

**Estimated Effort:** Small  
**Owner:** Data Governance

---

### Finding Application-016 — `Reference_Segment` Has No Physical Table

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | Medium |
| Guideline Rule | Modeling rule 2.11 |
| Evidence | Logical model: `Reference_Segment` (9 attrs); Physical: no `reference_segment` table |
| Affected Table | `silver.application.reference_segment` (required, missing) |
| Affected Column(s) | All |
| Confidence | 1.0 |

**Description:** The `Application` entity uses `Segment code` and `Segment description` but the reference table for segment codes has no physical implementation. Without the reference table, code canonicalization (SLV-004) cannot be enforced for segment codes.

**Remediation:** Create `reference_segment` physical table, or — preferably — reference the segment code from `silver.party` (where customer segmentation is maintained canonically).

**Estimated Effort:** Small  
**Owner:** Data Modeling / Data Engineering

---

### Finding Application-017 — `Reference_Facility` Has No Physical Table

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | Medium |
| Guideline Rule | Modeling rule 2.11 |
| Evidence | Logical model: `Reference_Facility` (14 attrs); Physical: no `reference_facility` table (though `reference_application_loan` exists with 1.7 MB and no logical entity) |
| Affected Table | `silver.application.reference_facility` (required, missing) |
| Affected Column(s) | All |
| Confidence | 0.90 |

**Description:** The `Application` entity references `Facility number`, `Facility type`, `Facility code`, and `Facility category id`, but the `Reference_Facility` table has no physical implementation. The `reference_application_loan` orphan table (1.7 MB) may be the intended implementation — if so, it requires alignment with the logical model.

**Remediation:** Clarify whether `reference_application_loan` is the intended physical implementation of `Reference_Facility`. If yes — rename and align with the logical model. If no — create `reference_facility` DDL.

**Estimated Effort:** Small  
**Owner:** Data Modeling

---

### Finding Application-018 — `Reference_Address` Is an Undefined Stub Entity

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | Medium |
| Guideline Rule | Modeling rule 2.11; `Application` entity uses "Address type" FK with no target |
| Evidence | Logical model: `Reference_Address` (1 attribute: None/null) — entity definition is empty; Physical: `reference_address_type` table exists (113 KB) |
| Affected Table | `silver.application.reference_address_type` (orphan, no entity) |
| Affected Column(s) | All |
| Confidence | 0.95 |

**Description:** `Reference_Address` is modelled as a single null attribute — it is an empty placeholder with no substance. Meanwhile, a physical `reference_address_type` table (113 KB with data) exists without a logical model counterpart. The `Application` entity uses "Address type" which presumably references this table. The gap between model and physical is a governance failure.

**Remediation:** Either (1) flesh out `Reference_Address` with proper attributes (address_type_code, address_type_description, is_active) matching the physical `reference_address_type` table, or (2) remove the stub entity and register the physical table under a formal entity name `Reference_Address_Type`.

**Estimated Effort:** Small  
**Owner:** Data Modeling

---

## 5. Critical Orphan Table Findings

### Finding Application-019 — `application_gold_fix` Is a Critical Architecture Violation

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | Modeling rule 5.1 — "Enforce Bronze → Silver → Gold → Semantic data flow... no layer may read from a downstream layer"; modeling rule 2.11 — all physical tables must have a logical entity |
| Evidence | Physical: `silver.application.application_gold_fix` (8 files, 284.9 MB); Logical: no entity; Description: null |
| Affected Table | `silver.application.application_gold_fix` |
| Affected Column(s) | All |
| Confidence | 0.95 |

**Description:** The table name `application_gold_fix` explicitly suggests this is data that was used to fix a Gold-layer issue by storing intermediate results in Silver. This is a critical violation of:
1. **Unidirectional data flow** — Gold data written back into Silver
2. **Naming convention** — "gold_fix" is not a valid Silver entity name pattern
3. **Governance** — 285 MB of data in a table with no logical entity, no description, no mapping

This table poses a data quality and audit risk: if Gold consumers are also reading from this Silver table directly, two independent data paths exist for the same data.

**Remediation:** (1) Immediately document what data this table contains and why it exists. (2) If it's a workaround fix, migrate the corrected data through proper Bronze → Silver → Gold pipeline and drop this table. (3) Apply a data catalog quarantine flag immediately. Do not allow this table to be referenced in any new pipeline.

**Estimated Effort:** Medium  
**Owner:** Data Engineering / Data Governance (urgent escalation required)

---

### Finding Application-020 — `dummy_reference_education` Is a Development Artifact in Production Silver

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | Modeling rule 2.11 — "Tables that exist without a logical model entry are a governance gap"; general governance principle |
| Evidence | Physical: `silver.application.dummy_reference_education` (2 files, 7.6 KB); Logical: no entity; Description: null |
| Affected Table | `silver.application.dummy_reference_education` |
| Affected Column(s) | All |
| Confidence | 1.0 |

**Description:** A table explicitly named `dummy_reference_education` exists in the Silver production layer. "Dummy" is unambiguous — this is a development/testing artifact that was never cleaned up. Its presence constitutes: (1) a governance failure, (2) a potential data quality contamination risk if any pipeline mistakenly reads from it, (3) a storage waste.

**Remediation:** Drop `silver.application.dummy_reference_education` immediately after confirming no downstream dependencies. Add a governance control to prevent `dummy_*` or `test_*` tables from being deployed to production Silver schemas.

**Estimated Effort:** Small  
**Owner:** Data Engineering / Data Governance

---

### Finding Application-021 — `loan_gl_account` Belongs in `silver.gl` Not `silver.application`

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | `sa/subject-areas.md` SA #19 Finance GL — "The Books: Chart of Accounts, General Ledger"; modeling rule 2.2 — subject-area isolation |
| Evidence | Physical: `silver.application.loan_gl_account` (1 file, 19.7 KB); Logical: no entity; Finance GL SA is SA #19 |
| Affected Table | `silver.application.loan_gl_account` |
| Affected Column(s) | All |
| Confidence | 0.95 |

**Description:** A GL account mapping table (`loan_gl_account`) is stored in the Application SA. GL account data is canonical to the Finance GL subject area (`silver.gl`). Having GL account data in Application SA creates a fragmented Chart of Accounts and breaks BCBS 239 lineage for any loan-to-GL reconciliation use cases.

**Remediation:** Move `loan_gl_account` to `silver.gl`. Create a logical model entity for it in the Finance GL SA. Update any pipelines referencing this table in Application SA.

**Estimated Effort:** Small  
**Owner:** Data Architecture / Finance GL data steward

---

### Finding Application-022 — `reference_application_ruleengine_cards*` Tables Are Unmodelled Operational Data

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | High |
| Guideline Rule | Modeling rule 2.11 — "Tables that exist without a model entry are governance gaps"; modeling rule 1.2 — "No business logic in Bronze; no raw data in Gold" (and by extension, no raw operational data in Silver) |
| Evidence | Physical: 7 `reference_application_ruleengine_cards*` tables totalling ~8.5 GB (capsbase: 7.4 GB, 1.54M files; applicantdetails: 331 MB; etc.); Logical: no entities |
| Affected Table | `reference_application_ruleengine_cards`, `reference_application_ruleengine_cards_applicantdetails`, `reference_application_ruleengine_cards_capsbase`, `reference_application_ruleengine_cards_decision_history`, `reference_application_ruleengine_cards_ibpsbase`, `reference_application_ruleengine_cards_nationality`, `reference_application_ruleengine_cards_rejectionhistory`, `reference_application_ruleengine_cards_rejectionreasonsbase`, `reference_application_ruleengine_cards_score_final` |
| Affected Column(s) | All |
| Confidence | 0.90 |

**Description:** Nine tables prefixed `reference_application_ruleengine_cards` collectively hold ~8.5 GB of data with no logical model, no descriptions, and no mapping documentation. The "capsbase" table alone has 1.54 million files — a clear sign of unbounded append-only operational ingestion, not a Silver reference table pattern. These appear to be raw decision engine outputs from a CAPS/IBPS scoring system. Naming these as "reference" tables is misleading — they are operational transactional data.

Additionally, the naming convention `reference_*` for these tables is incorrect — Silver entity tables should follow the `<entity>` naming pattern per NAM-003.

**Remediation:** (1) Immediately define the logical model for any of these tables that represent legitimate Silver entities. (2) For raw operational CAPS/IBPS data: move to Bronze layer or a dedicated operational store. (3) For DQ-validated scoring data needed in Silver: define proper logical entities (e.g., `card_application_score`, `card_application_decision`) with SK/BK and metadata columns. (4) Address the 1.54M-file fragmentation in `capsbase` with an `OPTIMIZE` compaction run.

**Estimated Effort:** Large  
**Owner:** Data Architecture / ETL / Risk domain steward

---

### Finding Application-023 — `application_loan` and `application_loan_ib` Have No Logical Entities

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | High |
| Guideline Rule | Modeling rule 2.11 — "Tables that exist without a model entry are governance gaps" |
| Evidence | Physical: `application_loan` (2 files, 24.8 MB), `application_loan_ib` (1 file, 25 MB); Logical: no entities for either |
| Affected Table | `silver.application.application_loan`, `silver.application.application_loan_ib` |
| Affected Column(s) | All |
| Confidence | 1.0 |

**Description:** Two loan application detail tables exist with significant data (25 MB each) but no corresponding logical model entities. The IB suffix suggests Islamic Banking. Per the SA taxonomy, Islamic Finance structures should use the `is_islamic_structure` flag rather than separate tables unless warranted by significantly different attributes. The correct approach is: (1) model `Application_Loan` as a formal entity, (2) if Islamic Banking requires structurally different attributes, use a companion table pattern (modeling rule 2.10).

**Remediation:** Define `Application_Loan` logical entity (and optionally `Application_Loan_IB` as a companion). Alternatively, merge into `Application_Feature` if loan features can be represented as features. Add to logical model, create data mappings, add SK/BK and audit columns.

**Estimated Effort:** Medium  
**Owner:** Data Modeling / Business

---

### Finding Application-024 — `application_form` Has No Logical Entity

| Field | Value |
|---|---|
| Priority | P1 |
| Criticality | Medium |
| Guideline Rule | Modeling rule 2.11 |
| Evidence | Physical: `silver.application.application_form` (1 file, 287 KB); Logical: no entity |
| Affected Table | `silver.application.application_form` |
| Affected Column(s) | All |
| Confidence | 1.0 |

**Description:** An `application_form` table exists with no logical model entity or description. This may contain raw form submission data — if so, it may be a Bronze artifact promoted without a Silver model.

**Remediation:** Define a logical entity for this data or, if it is raw form data, move to Bronze and define a proper Silver entity that captures only the canonicalized attributes.

**Estimated Effort:** Small  
**Owner:** Data Modeling

---

## 6. Missing Critical Entities Check

The following entities are **absent from both the logical model and physical structures** but are required by the subject-area definition and banking best practice:

| Missing Entity | Required Because | Priority | Industry Reference |
|---|---|---|---|
| `Application_Workflow_Event` | SA definition explicitly states "onboarding workflows and TAT tracking" — no entity captures workflow state transitions or TAT milestones | P0 | BIAN Product/Service Application Processing |
| `Application_Document` | KYC documents submitted with applications must be tracked for regulatory audit trail | P1 | BIAN Customer Document Handling |
| `Application_Decision` | Formal credit/service decision (approved/declined/referred) separate from the application header | P1 | BIAN Credit Assessment |
| `Application_Condition` | Pre-disbursement conditions (e.g., "submit salary certificate within 30 days") must be tracked | P2 | BIAN Product/Service Agreement |

---

### Finding Application-025 — Application_Workflow_Event Entity Missing (TAT Tracking Gap)

| Field | Value |
|---|---|
| Priority | P0 |
| Criticality | High |
| Guideline Rule | `sa/subject-areas.md` SA #16 — "onboarding workflows, service requests, and TAT tracking" |
| Evidence | SA taxonomy explicitly names TAT tracking as a business requirement; no workflow event entity exists in either logical model or physical structures |
| Affected Table | `silver.application.application_workflow_event` (required, missing) |
| Affected Column(s) | All |
| Confidence | 0.95 |

**Description:** The canonical SA description for Application includes "onboarding workflows and TAT tracking." No entity exists in the logical model or physical layer to capture workflow state transitions (e.g., Application Submitted → Under Review → Credit Decision → Documents Requested → Approved → Disbursed) or the timestamps between them that enable TAT calculation. This is a complete gap for a stated business use case. TAT metrics for onboarding were explicitly listed in the Business Feedback section of `sa/subject-areas.md`.

**Remediation:** Define `Application_Workflow_Event` entity with: `application_workflow_event_sk`, `application_workflow_event_bk`, `application_sk` (FK), `workflow_stage_code`, `event_timestamp TIMESTAMP`, `previous_stage_code`, `elapsed_minutes INTEGER`, `actor_id`, `channel_code`, plus standard metadata columns.

**Estimated Effort:** Medium  
**Owner:** Data Modeling / Business

---

## 7. Denormalization Register

| # | Entity | Denormalized Attribute | Source Entity | Classification | Justification Required | Finding |
|---|---|---|---|---|---|---|
| 1 | Application | Segment code + Segment description | Reference_Segment | **Unnecessary** | ❌ None documented | APP-002 (implied) |
| 2 | Application | Agency name | Reference_Agency | **Unnecessary** | ❌ None documented | Remove; use FK |
| 3 | Application | Application credit risk score | Application_Credit_Risk_Score | **Unnecessary** | ❌ Duplicate entity exists | APP-009 |
| 4 | Application | Total application Score | Application_Credit_Risk_Score | **Unnecessary** | ❌ Aggregation violates SLV-007 | APP-002 |
| 5 | Application | Total loan amount | Application_Loan | **Unnecessary** | ❌ Aggregation violates SLV-007 | APP-002 |
| 6 | Application | Total gross monthly income amount | Application_Customer | **Unnecessary** | ❌ Aggregation violates SLV-007 | APP-002 |
| 7 | Application | Total deposit balance amount | silver.account | **Unnecessary** | ❌ Cross-SA aggregation violates SLV-007 | APP-002 |
| 8 | Application | Total limit amount | Application_Loan | **Unnecessary** | ❌ Aggregation violates SLV-007 | APP-002 |
| 9 | Application | Total fair market value collateral amount | silver.collateral | **Unnecessary** | ❌ Cross-SA aggregation | APP-002 |
| 10 | Application_Customer | Race, Nationality, Gender, Education, Occupation, Marital status | silver.party | **Unnecessary** | ❌ Party SA is source of truth | APP-006 |
| 11 | Application_Customer | Asset liability ratio | Computed | **Unnecessary** | ❌ Derived ratio violates SLV-007 | — |
| 12 | Application_Customer | Credit bureau score | External / Party SA | **Acceptable** | ✅ Application-time snapshot needed | Retain with annotation |
| 13 | Application_Customer | Bank balance amount | silver.account | **Unnecessary** | ❌ Cross-SA at application time | Move to snapshot entity |

All denormalizations classified as **Unnecessary** must be removed. The two **Acceptable** co-locations (credit bureau score as application-time snapshot) must be documented with explicit data steward sign-off and a `_snapshot_date` companion column.

---

## 8. Guideline Compliance Summary

| Rule | Description | Status | Finding(s) |
|---|---|---|---|
| SLV-001 | `<entity>_sk` MD5 surrogate on every entity | 🔴 **Failed** — 0/28 entities | APP-001 |
| SLV-002 | `<entity>_bk` business key on every entity | 🔴 **Failed** — 0/28 entities | APP-001 |
| SLV-003 | SCD-2 default; incorrect column names/types | 🔴 **Failed** — wrong names (Business Start/End Date, CHAR(18)) | APP-004 |
| SLV-004 | No source-system codes; use `silver.reference.code_mapping` | 🔴 **Failed** — 12 reference tables duplicated locally | APP-012 |
| SLV-005 | DQ-gated writes; quarantine table | 🔴 **Failed** — no DQ status columns in logical model | APP-005 |
| SLV-006 | 3NF; no derived values/aggregations | 🔴 **Failed** — 7 aggregations in Application entity | APP-002 |
| SLV-007 | No pre-computed metrics | 🔴 **Failed** — scores and totals inline in Application | APP-002, APP-009 |
| SLV-008 | All metadata columns present | 🔴 **Partial** — `run_id`, `source_ingestion_date`, DQ columns absent; SCD columns wrong names/types | APP-004, APP-010 |
| SLV-009 | All timestamps UTC | 🟡 **Cannot verify** — all dates typed as STRING; no type enforcement possible | APP-003 |
| SLV-010 | Monetary amounts in DECIMAL(18,4) with `_amount_aed` suffix | 🔴 **Failed** — all monetary amounts typed as STRING without currency suffix | APP-003 |
| NAM-001 | snake_case, lowercase, no reserved words | 🔴 **Partial** — `IsLogicallyDeleted`, `Feature_Id`, `Business Start Date`, `Application Feature UOM Cd`, `Is Active Flag` violate snake_case | Multiple |
| NAM-003 | Silver entity table naming: `<entity>` | 🟡 **Partial** — core tables correct; `application_gold_fix`, `dummy_reference_education` violate convention | APP-019, APP-020 |
| NAM-005 | No SQL reserved words as identifiers | 🟢 **Pass** — no reserved words identified | — |
| DQ Gate 2 | PII masking (SHA-256) for national IDs | 🔴 **Failed** — no masking annotations on national_identifier columns | APP-008 |
| Modeling 2.1 | 3NF — no repeating groups | 🔴 **Failed** — Application entity has transitive dependencies (segment description inline) | APP-002 |
| Modeling 2.2 | Subject-area isolation | 🔴 **Failed** — reference tables from 4 SAs present in application | APP-012, APP-013, APP-014, APP-021 |
| Modeling 2.3 | SCD-2 via effective_from/to/is_current | 🔴 **Failed** — wrong names and types on all 6 core entities | APP-004 |
| Modeling 2.4 | Deterministic MD5 surrogate keys | 🔴 **Failed** — no SKs defined | APP-001 |
| Modeling 2.5 | Canonical code mapping | 🔴 **Failed** — local reference tables, no `silver.reference.code_mapping` usage documented | APP-012 |
| Modeling 2.6 | No aggregations in Silver | 🔴 **Failed** | APP-002 |
| Modeling 2.7 | 11 mandatory audit columns | 🔴 **Partial** — `run_id`, `source_ingestion_date`, DQ columns missing; names/types incorrect | APP-004, APP-010 |
| Modeling 2.11 | Erwin model before DDL | 🔴 **Failed** — 25 physical tables lack logical model entities | Multiple |
| Modeling 2.12 | Catalog registration | 🔴 **Failed** — 45 of 49 tables have null/missing descriptions | — |

**Overall Guideline Compliance: 1/24 rules passing (4%)**

---

## 9. Remediation Plan

### 9.1 Prioritized Action List

#### P0 — Immediate (This Sprint / Within 2 Weeks)

| # | Action | Tables/Entities | Effort | Owner |
|---|---|---|---|---|
| P0-1 | **Remove `application_gold_fix`** from Silver after impact analysis and data migration | `application_gold_fix` | M | Data Engineering |
| P0-2 | **Remove `dummy_reference_education`** immediately | `dummy_reference_education` | S | Data Engineering |
| P0-3 | **Move `loan_gl_account`** to `silver.gl` | `loan_gl_account` | S | Data Engineering + GL steward |
| P0-4 | **Add surrogate and business keys** to all 28 logical entities; update physical DDL | All 28 entities | L | Data Modeling + DBA |
| P0-5 | **Fix SCD-2 column names and types**: rename `Business Start Date` → `effective_from TIMESTAMP`, `Business End Date` → `effective_to TIMESTAMP`, `Is Active Flag` → `is_current BOOLEAN`; add `run_id`, `source_ingestion_date` | All 6 core entities | L | Data Modeling + ETL |
| P0-6 | **Create source-to-target mapping documents** for all 49 physical tables | All tables | L | ETL / Data Governance |
| P0-7 | **Apply PII masking** to `national_identifier`, `old_national_id` in ETL pipeline | `application_customer` | M | ETL + Security |
| P0-8 | **Add DQ columns** (`data_quality_status`, `dq_issues`) to all entities and implement DQ gate | All 6 core entities | M | ETL + Data Engineering |
| P0-9 | **Assign correct data types** to all attributes (replace STRING/CHAR(18) with DATE, TIMESTAMP, BOOLEAN, DECIMAL, INTEGER as appropriate) | All 28 logical entities | L | Data Modeling |
| P0-10 | **Add `Application_Workflow_Event` entity** for TAT tracking | New entity | M | Data Modeling + Business |

#### P1 — Next Sprint (2–6 Weeks)

| # | Action | Tables/Entities | Effort | Owner |
|---|---|---|---|---|
| P1-1 | **Migrate shared reference tables** to `silver.reference`: country, currency, gender, education, marital_status, occupation, customer_type | 7 tables | M | Data Architecture + ETL |
| P1-2 | **Migrate `reference_bank_branch`** to `silver.org` | 1 table | S | Data Architecture |
| P1-3 | **Migrate `reference_account_status`** to `silver.account` | 1 table | S | Data Architecture |
| P1-4 | **Remove aggregated attributes** from `Application` entity (7 total fields) | `application` | M | Data Modeling + ETL |
| P1-5 | **Remove Party attributes** from `Application_Customer`; add `party_sk` FK | `application_customer` | L | Data Modeling + Business |
| P1-6 | **Resolve `application_customer_new`** shadow table — formalize or drop | `application_customer_new` | S | Data Engineering |
| P1-7 | **Define logical entities** for `application_loan`, `application_loan_ib` | New entities | M | Data Modeling + Business |
| P1-8 | **Create `application_old_national_id`** physical table | New table | M | Data Engineering |
| P1-9 | **Create `reference_segment`** physical table or reference `silver.party` | New table | S | Data Modeling |
| P1-10 | **Create `reference_facility`** physical table or align `reference_application_loan` | 1 table | S | Data Modeling |
| P1-11 | **Define and model `reference_application_ruleengine_cards*`** tables — create logical entities or move to Bronze | 9 tables | L | Data Architecture + Risk |
| P1-12 | **Fix `Reference_Address` stub entity** — flesh out or remove | `reference_address_type` | S | Data Modeling |
| P1-13 | **Add catalog descriptions** to all 45 tables missing them | 45 tables | M | Data Governance |
| P1-14 | **Define FK relationships** in logical model for all cross-entity references | All entities | M | Data Modeling |

#### P2 — Backlog (6–12 Weeks)

| # | Action | Tables/Entities | Effort | Owner |
|---|---|---|---|---|
| P2-1 | **Add `Application_Document` entity** for document submission tracking | New entity | M | Data Modeling + Business |
| P2-2 | **Add `Application_Decision` entity** for formal credit decision | New entity | M | Data Modeling + Business |
| P2-3 | **Add `Application_Condition` entity** for pre-disbursement conditions | New entity | S | Data Modeling |
| P2-4 | **Resolve `Reference_Ethnicity`** — legal review, then implement or remove | Review + optional table | S | Data Governance |
| P2-5 | **Define and implement missing reference tables**: `reference_residential_status`, `reference_channel_type`, `reference_score_value`, `reference_security_type_code`, `reference_source_system`, `reference_interest_index`, `reference_loan_purpose` — add to logical model or migrate to `silver.reference` | 7 tables | M | Data Modeling |
| P2-6 | **Apply `OPTIMIZE` compaction** on `reference_application_ruleengine_cards_capsbase` (1.54M files) | 1 table | S | DBA / Data Engineering |
| P2-7 | **Register all Silver tables in data catalog (GDGC)** with steward, classification, and lineage | All 49 tables | L | Data Governance |
| P2-8 | **Implement `silver.reference.code_mapping` integration** for canonical code resolution across all code columns | All entities | L | ETL + Data Engineering |
| P2-9 | **Evaluate Islamic Banking split**: merge `application_loan_ib` into `application_loan` with `is_islamic_structure BOOLEAN` flag per SA taxonomy guidance | `application_loan_ib` | M | Data Architecture + Business |

### 9.2 Recommended Schedule

| Week | Focus |
|---|---|
| Week 1–2 | P0-1 to P0-3: Remove `gold_fix`, `dummy_ref_education`, migrate `loan_gl_account`. Governance escalation on `application_gold_fix` |
| Week 3–4 | P0-4, P0-9: Surrogate/business key design + type corrections in logical model |
| Week 5–6 | P0-5, P0-8: SCD-2 column renaming, audit column additions, DQ gate implementation |
| Week 7–8 | P0-6: Data mapping documentation (priority entities: application, application_customer, application_loan) |
| Week 9–10 | P0-7, P0-10: PII masking; Application_Workflow_Event entity design |
| Week 11–14 | P1-1 to P1-6: Reference table migrations, party attribute refactor, shadow table resolution |
| Week 15–20 | P1-7 to P1-14: Remaining P1 items |
| Week 21+ | P2 backlog items |

---

## Appendix A: Data Mapping Coverage Summary

| Mapping Dimension | Status |
|---|---|
| Data mapping files found | 0 of expected ≥1 |
| Logical attributes with documented source | 0 / 409 (0%) |
| Logical attributes with mapping available | 0 / 409 (0%) |
| CDE (Critical Data Elements) identified | 0 / 409 (0%) |
| Physical tables with documented lineage | 0 / 49 (0%) |

**Implication:** No Silver column in this subject area has a traceable lineage to a source system. All DQ rules are undefined. BCBS 239 Principle 2 is entirely unmet.

---

## Appendix B: Guideline Rule Citation Index

| Rule Code | Source Document | Full Text |
|---|---|---|
| SLV-001 | `guidelines/11-modeling-dos-and-donts.md` §2.4 | "Derive surrogate keys deterministically: `MD5(UPPER(TRIM(business_key)))` for single-component keys" |
| SLV-002 | `guidelines/11-modeling-dos-and-donts.md` §2.4 | "Every Silver entity table must have `<entity>_bk` as the natural/business key from source" |
| SLV-003 | `guidelines/11-modeling-dos-and-donts.md` §2.3 | "Track all changes with SCD Type 2: every Silver entity table must have `effective_from`, `effective_to`, `is_current`" |
| SLV-004 | `guidelines/11-modeling-dos-and-donts.md` §2.5 | "Resolve all source-system codes to canonical platform values using `silver.reference.code_mapping`. Do not store source-system-specific codes in Silver entity columns." |
| SLV-005 | `guidelines/01-data-quality-controls.md` §3.2 | "Failed records routed to quarantine / error tables rather than discarded. All DQ results must be logged and auditable." |
| SLV-006 | `guidelines/11-modeling-dos-and-donts.md` §2.1 | "Decompose entities into 3NF: every non-key attribute depends on the whole primary key and nothing but the primary key." |
| SLV-007 | `guidelines/11-modeling-dos-and-donts.md` §2.6 | "Do not pre-compute totals, averages, counts, ratios, or any derived metrics in Silver tables." |
| SLV-008 | `guidelines/11-modeling-dos-and-donts.md` §2.7 | "Include the following Silver technical audit columns on every Silver entity table: source_system_code, source_system_id, create_date, update_date, delete_date, is_active_flag, effective_from, effective_to, is_current, run_id, source_ingestion_date" |
| SLV-009 | `guidelines/01-data-quality-controls.md` §4 | "Time Zone Normalisation — All timestamps stored in UTC" |
| SLV-010 | `guidelines/01-data-quality-controls.md` §2 Gate 2 | "All monetary columns must use DECIMAL(18,4)" |
| NAM-001 | `guidelines/06-naming-conventions.md` | "Use snake_case for all identifiers. Use lowercase only — no mixed case." |
| NAM-003 | `guidelines/06-naming-conventions.md` §Table Naming | "Silver entity: `<entity>` (e.g., `customer`, `account`, `transaction`)" |
| NAM-005 | `guidelines/06-naming-conventions.md` | "Never use SQL reserved words as identifiers (e.g., `date`, `value`, `order`)." |
| DQ Gate 2 | `guidelines/01-data-quality-controls.md` §2 | "EID, Card Numbers, Phone Numbers, and Passport Numbers masked via SHA-256 unless stored in a restricted container" |

---

## Appendix C: Industry Reference Alignment

| Pattern | Standard | Application SA Status |
|---|---|---|
| Application entity boundary | BIAN "Product/Service Application" service domain | 🟡 Partially aligned; over-wide |
| Applicant Party reference | BIAN "Party Reference Data Management" | 🔴 Not implemented; party data duplicated |
| Credit decision recording | BIAN "Credit Assessment" | 🔴 Missing `Application_Decision` entity |
| Document management | BIAN "Customer Document Handling" | 🔴 Missing `Application_Document` entity |
| Workflow TAT tracking | BIAN "Product/Service Application Processing" | 🔴 Missing `Application_Workflow_Event` entity |
| Islamic Banking structures | SA taxonomy guidance: `is_islamic_structure` flag | 🔴 Separate `application_loan_ib` table exists instead |
| BCBS 239 Principle 2 | Data aggregation lineage requirements | 🔴 Zero lineage coverage |
| UAE PDPL Article 5 | Consent and PII processing basis | 🔴 No masking annotations; ethnicity entity present without legal basis documented |

---

**End of Report**  
*Assessment based on artifacts available as of 2026-04-28. Physical column-level metadata was not available in the CSV — findings regarding physical column types are based on logical model only. Actual physical DDL review is recommended as a follow-up.*
