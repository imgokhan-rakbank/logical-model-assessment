# Silver Layer — Subject Areas

## Design Principles

The Silver subject-area model applies:

- **BIAN (Banking Industry Architecture Network)** service domains — v11
- **IFW (Industry Framework for Banking)** from IBM / Teradata
- **BCBS 239** principles for risk data aggregation
- **IFRS 9** requirements (ECL, staging, collateral)
- **UAE/Gulf banking operations**: CBUAE regulatory reporting, UAEFTS/RTGS payments, AML/FATF requirements, full Balance Sheet and P&L coverage
- **Business feedback**: POS/channel data, onboarding TAT, card authorization audit, contact center volumes, branch CVM, pre-delinquent monitoring, collections workflow events

Each subject area has a distinct lifecycle and entity set, a clear business owner and data steward, an Erwin logical model, and registered DQ rules.

---

## Subject Area List

| # | Subject Area | Schema | Description | Priority |
|---|---|---|---|---|
| 1 | **Reference Data** | `silver.reference` | Master Lookups: ISO Currencies, Country Codes, FX Rates, Commodity Prices (Gold/Silver), Channel Master, and Industry Codes (NACE/SIC). | 1 (Critical) |
| 2 | **Party** | `silver.party` | Universal legal-entity model: individuals, corporates, SMEs, employees, guarantors, UBOs, and counterparties. Absorbs customer identity and KYC lifecycle. | 1 (Critical) |
| 3 | **Product** | `silver.product` | The Catalog: Definitions for CASA, Loans, Credit Cards, Trade Finance instruments, Insurance plans, and Fee/Interest schedules. | 2 (High) |
| 4 | **Contract** | `silver.contract` | Legal Agreements: Account Opening Contracts, Loan Agreements, Card Member Agreements, and Service Agreements. Stores agreed rates, terms, and signatures. | 3 (Medium) |
| 5 | **Account** | `silver.account` | Holding Containers: Unified Balance records for CASA, Term Deposits, Loan Accounts, and Card Balances. Tracks lifecycle status. | 1 (Critical) |
| 6 | **Deposit** | `silver.deposit` | CASA and Term Deposit Details: savings, current, and fixed-term accounts including interest/profit schedules, rollover instructions, and early-breakage events. | 2 (High) |
| 7 | **Card** | `silver.card` | Plastic & Revolving Details: Card Numbers (Masked), Expiry Dates, Embossing Names, Credit Limits, Billing Cycles, Authorizations, and Audit Log. | 2 (High) |
| **8** | **Lending** | `silver.lending` | **Top-level credit subject area. Encompasses three sub-areas:** | 1 (Critical) |
| 8a | ↳ **Loan** | `silver.loan` | Credit facilities: Loans, Islamic structures (Murabaha/Ijara/Musharaka), Repayment Schedules, Disbursements, and IFRS 9 ECL Staging. | 1 (Critical) |
| 8b | ↳ **Collateral** | `silver.collateral` | Security pledged against lending: Real Estate, Metals, Cash, Securities. Asset Valuations, LTV Ratios, and Lien/Charge Registrations. | 2 (High) |
| 8c | ↳ **Collections** | `silver.collection` | Recovery & Legal: Pre-Delinquent Monitoring, Delinquency Cases, Collection Actions, Restructures, Settlements, Legal Proceedings, Write-offs, and Recoveries. | 2 (High) |
| 9 | **Wealth** | `silver.wealth` | Investment Services: Mutual Funds, Equity Portfolios, Bond Holdings, and Advisory Mandates. Tracks off-balance sheet customer wealth. | 4 (Low) |
| 10 | **Insurance** | `silver.insurance` | Bancassurance: Insurance Policies, Beneficiaries, Premium Schedules, and Claims History. | Not in Scope |
| 11 | **Business Event** | `silver.event` | The Journey Spine: Logins, Complaints, Leads, Address Changes, Contact Center Interactions, Branch CVM Visits, and POS Events. Includes Channel and Location keys. | 3 (Medium) |
| 12 | **Transaction** | `silver.transaction` | The Financial Ledger: Atomic Double-Entry Postings, Card Settlements, POS Transactions, Fee Reversals, and Interest Capitalization. | 2 (High) |
| 13 | **Payments** | `silver.payment` | Messaging & Clearing: SWIFT, UAEFTS/CBUAE Clearing, ISO 8583 network data, Correspondent Banking, and Sanctions Screening. | 3 (Medium) |
| 14 | **Trade Finance** | `silver.trade_finance` | Documentary Instruments: Letters of Credit (LC), Bank Guarantees (BG), Bills of Exchange, and Shipping Documents. | 3 (Medium) |
| 15 | **Treasury** | `silver.treasury` | Bank Investments: Interbank Deals, Securities (Bonds/Stocks/Sukuk), Derivatives (Swaps/Forwards), and Money Market placements. | 4 (Low) |
| 16 | **Application** | `silver.application` | Customer requests for products or services: applicant demographics, requested amounts, onboarding workflows, service requests, and TAT tracking. | 4 (Low) |
| 17 | **Risk & Compliance** | `silver.risk` | Compliance Guardrails: KYC Levels, AML Risk Scores, Watchlist Hits, Customer Risk Ratings, FATCA/CRS flags, and Basel RWA. | 1 (Critical) |
| 18 | **Organization** | `silver.org` | Internal Bank Structure: Branches, Departments, Cost Centers, Regions, ATMs, and Physical Assets. Links Employees to their workplace. | 3 (Medium) |
| 19 | **Finance GL** | `silver.gl` | The Books: Chart of Accounts, General Ledger, Trial Balances, and Financial Period definitions. Full Balance Sheet and P&L coverage. | 1 (Critical) |

---

## Lending Hierarchy

Lending is the **top-level subject area** for all credit activity. It contains three sub-areas that share the `silver.lending` schema:

```
Lending (silver.lending)
├── Loan          — credit facilities, repayment schedules, IFRS 9 staging
├── Collateral    — pledged assets, valuations, LTV, lien registrations
└── Collections   — delinquency cases, collection actions, write-offs, recoveries
```

Collections is **part of Lending**, not a separate peer subject area. All three sub-areas share the same schema and are governed by the Credit / Risk data steward.

---

## Structural Decisions

| Decision | Rationale |
|----------|-----------|
| **Customer** is a Party role, not a separate SA | Avoids duplication of identity data; KYC and segmentation live in `silver.party` |
| **Collections** is under Lending | Collections lifecycle starts and ends within the lending credit relationship |
| **Islamic Finance** within Lending and Product | Sharia products use `is_islamic_product` flag and `islamic_structure_code` — no separate SA required |
| **Digital Channels** within Business Event | Digital behavioral events are captured as Business Events in `silver.event` |
| **Correspondent Banking** within Payments | Nostro/vostro accounts and correspondent limits are payment infrastructure |

---

## Business Feedback Addressed

| Feedback | Resolution |
|----------|-----------|
| POS data and channel data missing | `CARD_POS` type group in Transaction; POS events in Business Event |
| Onboarding and service requests with TAT missing | New **Application** subject area with workflow TAT tracking |
| Complete Balance Sheet and P&L coverage required | Finance GL SA covers full BS/P&L; GL account hierarchy supports roll-up |
| Card authorization & audit log missing | `card_authorization` and `card_audit_log` entities in Card SA |
| Contact center volumes (Calls/Chats/Prospects) | `contact_center_interaction` in Business Event SA |
| Branches — CVM Tokens/Walk-ins/Services | `branch_visit` with CVM token data in Business Event; branch master in Organization |
| Collections — Pre-Delinquent Accounts missing | `pre_delinquent_account` entity in Collections sub-area |
| Collections — workflow events (Restructure, Settlements, IBPS) | `collections_workflow_event` covering all workflow types |
| Collections — Document Archival (IBPS) | `collections_document` entity for IBPS metadata |
| Risk scoring in Collections | Risk score snapshot at case open stored in `delinquency_case` |
| All products in Collections | `facility_type_code` covers all 10 product types including RAK Business products |
