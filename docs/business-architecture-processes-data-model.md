# Credit Management System: Business Architecture, Processes and Data Model (Detailed Design)

| Item | Value |
| --- | --- |
| Document | Detailed business design |
| Version | 0.2 (draft): adds functional module design, sections 8–10 |
| Date | 2026-11-16 |
| Scope | Corporate, SME and retail credit; on-balance-sheet loans and off-balance-sheet facilities (guarantees, L/Cs) |
| Related | [Departments and Roles](departments-and-roles.md) |

This document defines the business design of the Credit Management System (CMS): what it is responsible for, how each business process runs step by step, and what data it keeps. It is written for business analysts, architects, developers and testers, and is the basis for functional specifications and the physical database design.

## Contents

1. [Conventions](#1-conventions)
2. [Business architecture](#2-business-architecture)
3. [Main business processes](#3-main-business-processes)
4. [Business data model](#4-business-data-model)
5. [State machines](#5-state-machines)
6. [Business events](#6-business-events)
7. [Business rules catalog](#7-business-rules-catalog)
8. [Functional module design (microservices)](#8-functional-module-design-microservices)
9. [Additional business processes](#9-additional-business-processes)
10. [Additional data entities](#10-additional-data-entities)
11. [Open questions](#11-open-questions)

---

## 1. Conventions

| Prefix | Meaning | Example |
| --- | --- | --- |
| `C-` | Business capability | C-04 Credit approval |
| `P-` | Business process | P-02 Corporate credit approval |
| `BR-` | Business rule | BR-LIM-003 Limit occupation check |
| `E-` | Business event | E-LOAN-DISBURSED |
| `ST-` | Status code | ST-APP-SUBMITTED |

- **Amounts** are stored as `decimal(20,2)` in the transaction currency, plus a reporting-currency equivalent where noted.
- **Dates** are `date` (business date). **Timestamps** are `timestamp with time zone`.
- **Required** column: `Y` = mandatory, `N` = optional, `C` = conditional (rule given in the description).
- **Maker-checker** means one user enters or proposes, and a different user with the right authority confirms.

---

## 2. Business architecture

### 2.1 Context

The CMS is the bank's system of record for **credit decisions, limits, collateral, contracts and credit risk status**. The core banking system remains the system of record for **accounts and accounting balances**.

```mermaid
flowchart TB
    classDef cms fill:#eef2ff,stroke:#6366f1,color:#111
    classDef side fill:#f3f4f6,stroke:#9ca3af,color:#111
    CH["<b>Channels</b><br/>RM workbench · Internet / mobile banking · Partner open API"]:::side
    CMS["<b>Credit Management System</b><br/>Pre-loan: customer, application, rating, approval<br/>Cross-cutting: limits, collateral, authority, pricing<br/>In-loan: contract, disbursement, servicing<br/>Post-loan: monitoring, classification, collection"]:::cms
    INT["<b>Internal systems</b><br/>Core banking · Payments · General ledger<br/>AML / sanctions · Data warehouse · HR / org directory"]:::side
    EXT["<b>External sources</b><br/>Credit bureaus · Company & property registries<br/>Court records · Valuation firms · News / market data"]:::side
    CH --> CMS
    CMS <-->|accounts, postings, screening,<br/>reporting feeds, org data| INT
    EXT -->|reports, records,<br/>valuations, signals| CMS
```

### 2.2 Capability map

```mermaid
flowchart TB
    classDef cms fill:#eef2ff,stroke:#6366f1,color:#111
    classDef side fill:#f3f4f6,stroke:#9ca3af,color:#111
    L1["<b>Customer & product</b><br/>C-01 Customer management · C-02 Product management"]:::cms
    L2["<b>Pre-loan</b><br/>C-03 Credit origination · C-04 Rating & scoring · C-05 Credit approval"]:::cms
    L3["<b>Cross-cutting credit controls</b><br/>C-06 Limit management · C-07 Collateral management<br/>C-08 Pricing management · C-09 Authority management"]:::cms
    L4["<b>In-loan</b><br/>C-10 Contract management · C-11 Disbursement management · C-12 Loan servicing"]:::cms
    L5["<b>Post-loan</b><br/>C-13 Monitoring & early warning · C-14 Asset quality & provisioning<br/>C-15 Collection & recovery"]:::cms
    L6["<b>Management & platform</b><br/>C-16 Portfolio & risk analytics · C-17 Reporting · C-18 Platform & control"]:::side
    L1 --> L2 --> L3 --> L4 --> L5 --> L6
```

### 2.3 Capability definitions

| ID | Capability | Level-2 capabilities | Business owner | Key outputs |
| --- | --- | --- | --- | --- |
| C-01 | Customer management | Onboarding & KYC; individual and organization profile; group & related parties; financial statements; external reports; customer 360° view | Front office | Customer record, group tree |
| C-02 | Product management | Product catalog; parameters; rate & fee schemes; eligibility rules; lifecycle (draft, launched, retired) | Product / Risk | Product definition |
| C-03 | Credit origination | Application intake; document checklist; due diligence report; financial analysis; credit proposal | Front office | Credit application |
| C-04 | Rating & scoring | Customer rating (PD); facility rating (LGD/EAD); retail scorecard; override; periodic re-rating | Risk | Rating record |
| C-05 | Credit approval | Routing; review; delegated approval; committee; conditions; reconsideration | Credit Approval | Approval decision |
| C-06 | Limit management | Limit tree; activation; occupation & release; freeze; adjustment; concentration caps | Risk / Credit Admin | Active limit, limit transactions |
| C-07 | Collateral management | Registration; valuation; allocation; perfection; insurance; monitoring; release | Credit Admin | Collateral, LTV |
| C-08 | Pricing management | Base rates; pricing grid by rating; risk-adjusted pricing; pricing exceptions | Risk / Treasury | Approved price |
| C-09 | Authority management | Authority matrix; delegation; temporary authority; authority audit | Risk / CCO | Required approval level |
| C-10 | Contract management | Templates; generation; signing; amendment; archive | Credit Admin / Legal | Effective contract |
| C-11 | Disbursement management | Drawdown request; conditions checklist; disbursement approval; entrusted payment | Credit Admin | Loan account |
| C-12 | Loan servicing | Schedule; accrual; repayment; prepayment; extension; rate reset | Operations | Balances, schedule |
| C-13 | Monitoring & early warning | Scheduled reviews; covenant tests; warning signals; watch list | Monitoring | Monitoring report, actions |
| C-14 | Asset quality & provisioning | Risk classification; IFRS 9 staging; ECL; impairment assessment | Risk / Finance | Classification, ECL |
| C-15 | Collection & recovery | Delinquency buckets; strategies; actions; litigation; enforcement; restructuring; write-off | Collections / Legal | Recovery, write-off |
| C-16 | Portfolio & risk analytics | Concentration; stress testing; RWA; limit utilization; vintage analysis | Risk | Risk reports |
| C-17 | Reporting | Regulatory reports; management dashboards; RAROC | Finance / Reporting | Reports |
| C-18 | Platform & control | Workflow; rules; models; users, roles & data scope; audit; documents; notifications; batch | IT | Platform services |

### 2.4 Business domains (module boundaries)

Section 8 turns these domains into 17 deployable functional modules (M01–M17). Each domain owns its data. Other domains read it through services or events and do not update it directly.

| Domain | Owns (master data) | Consumes from |
| --- | --- | --- |
| Party | Customer, group, related party, financial statement | AML, credit bureau, registry |
| Product | Product, parameter, rate & fee scheme | — |
| Origination | Application, facility request, due diligence report | Party, Product, Rating |
| Rating | Rating model, rating record | Party (financials), external data |
| Approval | Approval task, opinion, decision, condition | Origination, Authority |
| Authority | Authority matrix, delegation | Org directory |
| Limit | Limit, limit transaction, freeze | Approval, Contract, Loan |
| Collateral | Collateral, valuation, link, registration, insurance | Party, valuation firms |
| Contract | Loan, security and guarantee contracts | Approval, Limit, Collateral |
| Loan | Drawdown, schedule, transaction, accrual | Contract, Limit, core banking |
| Monitoring | Monitoring task, report, warning signal, covenant | Party, Loan, external data |
| Asset quality | Classification, ECL result | Loan, Rating, Monitoring |
| Collection | Delinquency case, action, promise, litigation, write-off | Loan, Collateral |

### 2.5 Organization-to-capability mapping (RACI)

R = responsible, A = accountable, C = consulted, I = informed.

| Capability | RM / Front | Credit Approval | Risk Mgmt | Credit Admin | Operations | Collections | Legal | Finance | Compliance |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| C-01 Customer | R/A | I | I | I | — | — | — | — | C |
| C-03 Origination | R/A | C | I | — | — | — | — | — | — |
| C-04 Rating | R | C | A | — | — | — | — | — | — |
| C-05 Approval | C | R/A | C | I | — | — | C | — | C |
| C-06 Limit | I | C | A | R | — | — | — | — | — |
| C-07 Collateral | C | C | I | R/A | — | — | C | — | — |
| C-10 Contract | C | I | — | R/A | — | — | C | — | — |
| C-11 Disbursement | C | I | — | R/A | C | — | — | I | — |
| C-12 Servicing | I | — | — | C | R/A | — | — | I | — |
| C-13 Monitoring | R | I | A | — | — | — | — | — | — |
| C-14 Asset quality | C | C | R/A | — | — | — | — | R | — |
| C-15 Collection | C | I | C | — | — | R/A | R | I | — |

### 2.6 Architecture principles

| # | Principle | Design consequence |
| --- | --- | --- |
| AP-1 | One credit view per customer and group | Every exposure type (loan, guarantee, L/C, bill, card) occupies the same limit tree |
| AP-2 | Approval, contract and drawdown are separate objects | Three entities, each with its own status; one approval can back many contracts, one contract many drawdowns |
| AP-3 | Segregation of duties is enforced by the system | Role-conflict rules; the same user cannot act twice on one case in incompatible roles |
| AP-4 | Business behavior is configured, not coded | Products, routes, authority, rules, checklists and models are versioned parameter data |
| AP-5 | Every change is traceable | Immutable audit log; history tables for rating, classification, valuation, limit |
| AP-6 | Events drive downstream updates | Key events published to a message bus; reports and other domains subscribe |
| AP-7 | Core banking is the accounting master | CMS reconciles balances daily and never posts accounting entries outside the agreed interface |

---

## 3. Main business processes

### 3.1 Process map

```mermaid
flowchart LR
    P01[P-01 Customer<br/>onboarding] --> P02[P-02 Corporate<br/>application & approval]
    P01 --> P03[P-03 Retail<br/>auto-decision]
    P04[P-04 Rating] -.supports.-> P02
    P04 -.supports.-> P03
    P02 --> P05[P-05 Limit<br/>activation]
    P03 --> P05
    P06[P-06 Collateral<br/>management] -.supports.-> P05
    P05 --> P07[P-07 Contract<br/>signing]
    P07 --> P08[P-08 Disbursement]
    P08 --> P09[P-09 Repayment &<br/>servicing]
    P09 --> P10[P-10 Loan<br/>changes]
    P09 --> P11[P-11 Monitoring &<br/>early warning]
    P11 --> P12[P-12 Classification<br/>& provisioning]
    P09 --> P13[P-13 Collection]
    P13 --> P14[P-14 Restructuring,<br/>NPL & write-off]
    P09 --> P15[P-15 Closure &<br/>collateral release]
```

| ID | Process | Trigger | Main actors | Output | Target time |
| --- | --- | --- | --- | --- | --- |
| P-01 | Customer onboarding & maintenance | New customer; data change | RM, Compliance, reviewer | Active customer | 1 business day |
| P-02 | Corporate credit application & approval | Financing need; renewal; increase | RM, analyst, approvers, committee | Approval decision | 5–15 business days by level |
| P-03 | Retail credit application (auto-decision) | Online or branch application | Customer, system, underwriter | Approve / decline / refer | Seconds (auto); 1 day (manual) |
| P-04 | Rating & re-rating | New application; annual review; trigger event | RM, risk analyst | Rating record | Within P-02 |
| P-05 | Limit management | Approval; change; freeze; expiry | Credit Admin, risk | Limit status change | Same day |
| P-06 | Collateral management | Collateral offered; revaluation due; release | Credit Admin, valuer, legal | Perfected collateral | 5–20 days (registration) |
| P-07 | Contract signing | Approval with active limit | Credit Admin, legal, customer | Effective contract | 1–3 days |
| P-08 | Disbursement | Drawdown request | Customer, RM, Credit Admin | Disbursed loan | Same day |
| P-09 | Repayment & servicing | Due date; customer payment | System, operations | Updated balances | Daily batch |
| P-10 | Loan changes | Customer request | RM, approver, Credit Admin | Amended loan | 3–10 days |
| P-11 | Monitoring & early warning | Schedule; warning signal | RM, monitoring officer | Monitoring report; actions | By signal severity |
| P-12 | Classification & provisioning | Month-end; trigger event | System, risk, finance | Classification; ECL | Month-end + 5 days |
| P-13 | Collection | Overdue | Collectors, RM | Repayment or escalation | By DPD bucket |
| P-14 | Restructuring, NPL & write-off | Distress; DPD 90+ | Special assets, legal, approvers | Restructured / sold / written-off loan | Case by case |
| P-15 | Closure & collateral release | Full repayment | System, Credit Admin | Closed loan; released collateral | 3 days |

---

### 3.2 P-01 Customer onboarding

```mermaid
flowchart TD
    A[RM: create customer] --> B{Duplicate check}
    B -- duplicate found --> B1[Open existing record]
    B -- none --> C[System: sanctions / PEP / AML screening]
    C --> D{Hit?}
    D -- yes --> E[Compliance review]
    E -- rejected --> X[Customer rejected]
    E -- cleared --> F
    D -- no --> F[RM: profile, shareholders, management,<br/>group & related parties]
    F --> G[RM: financial statements]
    G --> H[System: credit bureau & registry query]
    H --> I[Reviewer: maker-checker approval]
    I -- returned --> F
    I -- approved --> J[Customer active]
```

| Step | Actor | Activity | Input | Output | Rules / controls | Exceptions |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | RM | Create customer | ID documents, registration certificate | Draft customer | BR-CUS-001 unique key = ID type + ID number | Duplicate → reuse existing customer |
| 2 | System | Screen sanctions, PEP and AML lists | Name, ID, nationality, shareholders | Screening result | BR-CUS-002 any hit blocks the process | Screening service down → hold, retry |
| 3 | Compliance | Review hits | Screening result | Cleared / rejected | Decision and reason recorded | Rejection ends the process |
| 4 | RM | Maintain profile, group and related parties | Shareholder structure | Group links | BR-CUS-003 control ≥ 50% or de facto control → same group | Unclear control → risk decides |
| 5 | RM | Enter or import financial statements | Audited / management accounts | Financial statement | BR-CUS-004 balance sheet must balance | Unaudited → flag |
| 6 | System | Query credit bureau and registries | Customer consent | External report | Consent must be on file | No record → note "thin file" |
| 7 | Reviewer | Approve record | Complete customer | Active customer | Maker ≠ checker | Returned → back to step 4 |

---

### 3.3 P-02 Corporate credit application and approval

```mermaid
sequenceDiagram
    autonumber
    actor RM
    participant SYS as CMS
    actor AN as Credit analyst
    actor AP as Approver(s)
    actor CC as Credit committee
    RM->>SYS: Create application (type, facilities, amount, tenor, purpose)
    RM->>SYS: Attach due diligence report and documents
    RM->>SYS: Request rating
    SYS-->>RM: Rating result (PD grade)
    RM->>SYS: Propose limit, pricing, collateral, conditions
    RM->>SYS: Submit
    SYS->>SYS: Completeness check, policy rules, authority routing
    SYS->>AN: Review task
    alt incomplete
        AN-->>RM: Return with comments
    else complete
        AN->>SYS: Independent review opinion
        SYS->>AP: Approval task(s) by authority matrix
        alt within approver authority
            AP->>SYS: Decision
        else above authority
            AP->>SYS: Recommend and escalate
            SYS->>CC: Committee agenda item
            CC->>SYS: Vote and minutes
        end
    end
    SYS-->>RM: Approval notice with conditions
    SYS->>SYS: Create limit (pending activation)
```

**Step detail**

| Step | Actor | Activity | Rules / controls |
| --- | --- | --- | --- |
| 1 | RM | Create application, choosing type: new, renewal, increase, change, one-off | BR-APP-001 one open application per customer per facility type |
| 2 | RM | Add facility requests: product, amount, currency, tenor, repayment, purpose | Product eligibility rules (C-02) |
| 3 | RM | Complete due diligence report from template | Mandatory sections by customer segment |
| 4 | RM / System | Rating: model computes score; RM may propose override | BR-RAT-002 override needs reason and risk approval |
| 5 | RM | Propose collateral and guarantees from collateral pool or new entries | Collateral must exist in C-07 before approval |
| 6 | RM | Propose pricing | BR-PRC-001 price below grid floor = pricing exception, raises approval level |
| 7 | RM | Submit | All mandatory fields and documents present |
| 8 | System | Run policy rules: industry caps, related-party, large-exposure, blacklist | Rule hits shown to all approvers |
| 9 | System | Determine approval route from authority matrix | BR-AUT-001 (see 7) |
| 10 | Analyst | Independent review; write opinion | Analyst cannot edit the proposal |
| 11 | Approvers | Sequential or parallel approval | BR-APP-005 each approver acts once; no self-approval |
| 12 | Committee | Vote; secretary records minutes | Quorum and majority rules configurable |
| 13 | System | Issue decision; create pending limit | BR-APP-007 approval valid 180 days unless activated |

**Decision outcomes:** approve; approve with conditions; approve with amended terms; reject; return for more information. A rejected customer may apply again after a cooling-off period (configurable).

**Approval conditions** are of three types, each tracked to closure:
- **Pre-contract**: must be met before contract signing.
- **Pre-disbursement**: must be met before each or the first drawdown.
- **Post-disbursement**: covenants and follow-up items, monitored in P-11.

---

### 3.4 P-03 Retail credit application (automated decision)

```mermaid
flowchart TD
    A[Application received] --> B[Identity verification]
    B --> C[Anti-fraud checks:<br/>device, blacklist, velocity]
    C -- fraud suspected --> R1[Decline / fraud queue]
    C -- pass --> D[Data collection:<br/>bureau, income, internal history]
    D --> E[Policy rules:<br/>age, residency, DTI/DSR, exclusions]
    E -- hard fail --> R2[Decline with reason code]
    E -- pass --> F[Scorecard: score and risk band]
    F --> G{Decision matrix}
    G -- approve --> H[Assign limit and price by band]
    G -- refer --> M[Manual underwriter queue]
    G -- decline --> R3[Decline with reason code]
    M -- approve --> H
    M -- decline --> R3
    H --> I[E-contract signing]
    I --> J[Automatic disbursement]
```

**Decision matrix example**

| Score band | Policy result: clean | Policy result: soft warning |
| --- | --- | --- |
| A (lowest risk) | Approve | Approve |
| B | Approve | Refer |
| C | Refer | Refer |
| D | Decline | Decline |

Rules: BR-RET-001 DTI (total monthly debt ÷ monthly income) must not exceed the product maximum. BR-RET-002 decline reasons must be recorded as standard codes for customer notification and model monitoring.

---

### 3.5 P-04 Rating and re-rating

| Step | Activity | Rule |
| --- | --- | --- |
| 1 | Select model by segment (large corporate, SME, real estate developer, financial institution, retail) | Model version effective on the rating date |
| 2 | Quantitative factors from financial statements (leverage, coverage, liquidity, profitability) | Statements no older than 18 months |
| 3 | Qualitative factors entered by RM (management, market position, industry outlook) | Each answer from a fixed list |
| 4 | Model computes score → grade → PD | Grade scale fixed per model |
| 5 | Adjustments: group support, country ceiling, warning signals | Rules-based, logged |
| 6 | Override (optional) | BR-RAT-002 up to 2 notches with risk approval; more needs CRO |
| 7 | Approval by risk analyst | Maker ≠ checker |
| 8 | Rating becomes effective | Re-rating at least yearly; trigger events force re-rating |

---

### 3.6 P-05 Limit management

```mermaid
flowchart TD
    A[Approval decision] --> B[Limit created: Pending]
    B --> C{Pre-contract conditions met<br/>and contract signed?}
    C -- yes --> D[Limit Active]
    C -- no, validity expired --> E[Limit Expired]
    D --> F[Occupy on drawdown / guarantee / L/C]
    F --> G[Release on repayment<br/>if revolving]
    G --> D
    D --> H[Freeze by warning signal<br/>or manual decision]
    H --> I[Unfreeze with approval]
    I --> D
    D --> J[Adjust: increase / reduce / extend<br/>via new approval]
    J --> D
    D --> K[End date reached → Expired]
    D --> L[Cancelled → Closed]
```

**Occupation check (BR-LIM-003)** runs for every drawdown or off-balance issuance:

1. Convert the amount into the limit currency at the current rate.
2. Walk up the limit tree (sub-limit → facility → customer → group). At every level, `available_amount ≥ requested_amount`.
3. No level is frozen, expired or outside its validity dates.
4. Product is in the limit's product scope.
5. Industry, country and single-borrower concentration caps are not exceeded.
6. Occupy all levels in one transaction; if any check fails, nothing is occupied.

**Occupation weight:** off-balance-sheet items can occupy at a credit conversion factor (for example, a performance guarantee at 50%), configurable per product.

---

### 3.7 P-06 Collateral management

| Step | Actor | Activity | Rules |
| --- | --- | --- | --- |
| 1 | RM / Credit Admin | Register collateral and owner | Owner must be a customer record (third-party owners included) |
| 2 | Valuer | Appraise: market value, valuation date, method | BR-COL-001 external valuer from approved panel above threshold |
| 3 | System | Apply haircut by collateral type → eligible value | Haircut table versioned |
| 4 | Credit Admin | Link to limit or contract with allocated value and lien rank | BR-COL-003 sum of allocations ≤ eligible value |
| 5 | Credit Admin / Legal | Perfect: register mortgage or pledge; store originals; insurance with bank as loss payee | Registration status must be "registered" before disbursement unless waived |
| 6 | System | Monitor: revaluation due, insurance expiry, registration expiry, LTV breach | BR-COL-005 revaluation frequency by type |
| 7 | Credit Admin | Release after secured exposure is repaid | Approval required; registration cancelled |

**Revaluation frequency (example)**

| Collateral type | Haircut | Revaluation |
| --- | --- | --- |
| Cash deposit (same currency) | 0% | Daily balance check |
| Government bonds | 2–5% | Daily mark-to-market |
| Listed shares | 30–50% | Daily mark-to-market |
| Residential real estate | 20–30% | Yearly, or index-based |
| Commercial real estate | 30–40% | Yearly |
| Receivables | 30–50% | Monthly |
| Machinery and equipment | 40–60% | Yearly |

---

### 3.8 P-07 Contract signing

| Step | Actor | Activity | Rules |
| --- | --- | --- | --- |
| 1 | Credit Admin | Generate contract from template; system fills terms from the approval | BR-CON-001 contract amount, tenor, price and collateral must match the approval |
| 2 | Legal | Review only if clauses deviate from the template | Deviations highlighted |
| 3 | Credit Admin | Generate security contracts (mortgage, pledge) and guarantee contracts | One per collateral or guarantor |
| 4 | Customer and bank | Sign, on paper or by e-signature | Signatory authority checked against customer records |
| 5 | Credit Admin | Record signing; contract becomes effective | Pre-contract conditions closed |
| 6 | System | Activate limit if all activation conditions are met | E-LIMIT-ACTIVATED |

---

### 3.9 P-08 Disbursement

```mermaid
sequenceDiagram
    autonumber
    actor C as Customer / RM
    participant SYS as CMS
    actor MK as Credit Admin (maker)
    actor CK as Supervisor (checker)
    participant CORE as Core banking
    C->>SYS: Drawdown request (contract, amount, date, beneficiary)
    SYS->>SYS: Check contract effective, limit available, no freeze
    SYS->>MK: Disbursement task with checklist
    MK->>SYS: Confirm checklist items and documents
    SYS->>CK: Approval task
    CK->>SYS: Approve
    SYS->>SYS: Occupy limit (all levels)
    SYS->>CORE: Create loan account and disburse
    CORE-->>SYS: Account number and value date
    SYS->>SYS: Generate repayment schedule
    SYS-->>C: Disbursement notice
```

**Pre-disbursement checklist (configurable per product)**

- [ ] Contract and security contracts signed and effective
- [ ] Collateral registered (or waiver approved)
- [ ] Insurance in force with bank as loss payee
- [ ] All pre-disbursement approval conditions closed
- [ ] Use of funds evidence (invoice, purchase contract) for entrusted payment
- [ ] Customer not on the watch list with a disbursement block
- [ ] KYC still valid

Rules: BR-DIS-001 maker ≠ checker. BR-DIS-002 entrusted payment is required above a product-specific amount: funds go directly to the supplier. BR-DIS-003 if core banking posting fails, the limit occupation is reversed.

---

### 3.10 P-09 Repayment and loan servicing

**Daily batch (end of day)**

| Order | Job | Description |
| --- | --- | --- |
| 1 | Rate reset | Apply new base rate for floating loans on reset dates |
| 2 | Interest accrual | Daily interest on outstanding principal (day count per product, e.g. ACT/365) |
| 3 | Auto-debit | Collect installments due from the repayment account |
| 4 | Overdue marking | Unpaid installments become overdue; DPD increments |
| 5 | Penalty interest | Penalty rate on overdue principal and interest |
| 6 | Limit release | Release repaid principal on revolving limits |
| 7 | Event publish | E-LOAN-REPAID, E-LOAN-OVERDUE |
| 8 | Reconciliation | Compare balances with core banking; raise breaks |

**Repayment allocation order (BR-SRV-001, configurable):** fees → penalty interest → overdue interest → overdue principal → current interest → current principal.

**Repayment methods:** equal installments (annuity), equal principal, interest-only with bullet, balloon, custom schedule.

---

### 3.11 P-10 Loan changes

| Change type | Needs approval | Treated as restructuring if borrower is in difficulty |
| --- | --- | --- |
| Extension of maturity | Yes | Yes |
| Rescheduling (installment amounts or dates) | Yes | Yes |
| Rate change | Yes (pricing authority) | If rate is below market |
| Repayment account change | No (maker-checker only) | No |
| Prepayment | No, unless fee waiver | No |
| Collateral substitution | Yes | No |
| Guarantor change | Yes | No |

BR-CHG-001 a change that eases terms for a borrower with DPD > 0 or a "Special mention" or worse classification is flagged as restructuring (P-14).

---

### 3.12 P-11 Monitoring and early warning

```mermaid
flowchart TD
    S1[Scheduled task:<br/>first check, periodic review,<br/>annual renewal] --> T[Monitoring task]
    S2[Warning signal:<br/>internal or external] --> G{Severity}
    G -- red --> T1[Task due in 3 days,<br/>auto-freeze limit]
    G -- orange --> T2[Task due in 10 days]
    G -- yellow --> T3[Task due in 30 days]
    T1 --> T
    T2 --> T
    T3 --> T
    T --> I[RM / monitoring officer investigates]
    I --> D{Action}
    D --> A1[No action, close]
    D --> A2[Add to watch list]
    D --> A3[Freeze or reduce limit]
    D --> A4[Require more collateral]
    D --> A5[Downgrade rating / classification]
    D --> A6[Early recall]
    A2 --> R[Monitoring report approved by risk]
    A3 --> R
    A4 --> R
    A5 --> R
    A6 --> R
    A1 --> R
```

**Warning signal catalog (examples)**

| Code | Signal | Source | Default severity |
| --- | --- | --- | --- |
| EW-01 | Any installment overdue > 7 days | Loan | Yellow |
| EW-02 | Overdue > 30 days | Loan | Orange |
| EW-03 | Account inflows down > 50% vs 3-month average | Core banking | Orange |
| EW-04 | Rating downgrade by ≥ 2 notches | Rating | Orange |
| EW-05 | New lawsuit as defendant above threshold | Court records | Red |
| EW-06 | Default at another bank | Credit bureau | Red |
| EW-07 | Covenant breach | Covenant test | Orange |
| EW-08 | Collateral LTV above maximum | Collateral | Orange |
| EW-09 | Key person change (CEO, major shareholder) | Registry | Yellow |
| EW-10 | Negative news (fraud, bankruptcy, sanctions) | News | Red |

**Scheduled monitoring frequency**

| Classification / risk | Periodic review |
| --- | --- |
| Normal, low risk | Every 12 months |
| Normal, medium risk | Every 6 months |
| Special mention / watch list | Every 3 months |
| Non-performing | Monthly (case review) |

---

### 3.13 P-12 Risk classification and provisioning

| Step | Actor | Activity |
| --- | --- | --- |
| 1 | System | Month-end batch proposes classification per loan from DPD, restructuring flag, rating and signals |
| 2 | System | Customer-level alignment: all loans of a customer take the worst class (BR-CLS-002, configurable) |
| 3 | RM | Propose adjustments with reasons |
| 4 | Risk | Approve adjustments; upgrades need stricter approval than downgrades |
| 5 | System | Determine IFRS 9 stage and compute ECL = PD × LGD × EAD (scenario-weighted) |
| 6 | Risk | Individual impairment assessment for large Stage 3 exposures (discounted recovery cash flows) |
| 7 | Finance | Approve provisions; post to GL |
| 8 | System | Lock the period; publish E-CLASS-CHANGED and E-ECL-POSTED |

**Classification mapping (example; local rules apply)**

| DPD / condition | Five-category | IFRS 9 stage |
| --- | --- | --- |
| 0, no weakness | Normal | Stage 1 |
| 1–30, or weakness identified | Normal / Special mention | Stage 1 or 2 |
| 31–90, or significant increase in credit risk | Special mention | Stage 2 |
| 91–180 | Substandard | Stage 3 |
| 181–360 | Doubtful | Stage 3 |
| > 360, or recovery unlikely | Loss | Stage 3 |
| Restructured, in probation | At least Substandard (configurable) | Stage 2 or 3 |

---

### 3.14 P-13 Collection

```mermaid
flowchart LR
    A[DPD 1–7<br/>SMS / app reminder] --> B[DPD 8–30<br/>Call center]
    B --> C[DPD 31–60<br/>Field visit,<br/>demand letter,<br/>limit frozen]
    C --> D[DPD 61–90<br/>Senior collector,<br/>guarantor notice]
    D --> E[DPD 90+<br/>Special assets team,<br/>legal action]
    E --> F{Recovery route}
    F --> F1[Restructure]
    F --> F2[Collateral enforcement]
    F --> F3[Guarantor claim]
    F --> F4[Litigation]
    F --> F5[NPL sale]
    F --> F6[Write-off]
```

| Item | Design |
| --- | --- |
| Case creation | Automatic at DPD 1; one case per customer, covering all overdue loans |
| Strategy | Assigned by segment × DPD bucket × risk score; configurable |
| Actions | Call, SMS, letter, visit, legal notice; each logged with outcome |
| Payment promise | Date and amount; broken promise escalates the case |
| Agency assignment | Late-stage cases may go to external agencies; commission tracked |
| Closure | Overdue fully cleared, restructured, sold or written off |

---

### 3.15 P-14 Restructuring, NPL and write-off

| Route | Key steps | Approval |
| --- | --- | --- |
| Restructuring | Viability analysis → new terms → approval → amendment contract → restructured flag → probation (e.g. 12 months) | Per authority matrix, one level above normal |
| Collateral enforcement | Legal notice → auction or private sale → proceeds allocated to the loan | Special assets head |
| Litigation | Case filed → hearings → judgment → enforcement → recovery | Legal head |
| NPL sale | Portfolio selection → valuation → bidding → sale agreement → derecognition | Board or delegated committee |
| Write-off | Evidence that recovery is unlikely → provision 100% → approval → off-balance tracking → continued recovery | High level (CRO / board), per local rules |

BR-WOF-001 written-off accounts stay in the CMS off-balance-sheet with continued collection and recovery tracking.

---

### 3.16 P-15 Closure and collateral release

| Step | Activity | Rule |
| --- | --- | --- |
| 1 | Final repayment received; loan balance = 0 | All fees and interest settled |
| 2 | Loan closed; limit released or closed | Non-revolving limit closes |
| 3 | Check whether collateral secures other exposures | BR-COL-007 release only if no other secured exposure |
| 4 | Release approval | Maker-checker |
| 5 | Cancel registration; return original documents | Customer receipt recorded |
| 6 | Archive the credit file | Retention period per policy (e.g. 10 years) |

---

## 4. Business data model

### 4.1 Subject areas

| Code | Subject area | Core entities |
| --- | --- | --- |
| PTY | Party | CUSTOMER, INDIVIDUAL, ORGANIZATION, CUSTOMER_GROUP, GROUP_MEMBER, RELATED_PARTY, FINANCIAL_STATEMENT, FIN_STMT_ITEM, EXTERNAL_REPORT |
| PRD | Product | PRODUCT, PRODUCT_PARAM, RATE_SCHEME, FEE_SCHEME |
| APP | Application & approval | CREDIT_APPLICATION, FACILITY_REQUEST, APPROVAL_TASK, APPROVAL_OPINION, COMMITTEE_MEETING, COMMITTEE_VOTE, APPROVAL_DECISION, APPROVAL_CONDITION |
| RAT | Rating | RATING_MODEL, RATING_RECORD, RATING_FACTOR_SCORE |
| AUT | Authority | AUTHORITY_RULE, AUTHORITY_DELEGATION |
| LIM | Limit | CREDIT_LIMIT, LIMIT_TXN, LIMIT_FREEZE |
| COL | Collateral | COLLATERAL, COLLATERAL_VALUATION, COLLATERAL_LINK, COLLATERAL_REGISTRATION, INSURANCE_POLICY, GUARANTEE |
| CON | Contract | LOAN_CONTRACT, SECURITY_CONTRACT, CONTRACT_AMENDMENT |
| LON | Loan | LOAN, REPAYMENT_SCHEDULE, LOAN_TXN, INTEREST_ACCRUAL, RATE_HISTORY |
| MON | Monitoring | MONITORING_TASK, MONITORING_REPORT, WARNING_SIGNAL, WATCH_LIST, COVENANT, COVENANT_TEST |
| AQ | Asset quality | RISK_CLASSIFICATION, ECL_RESULT |
| CLC | Collection | DELINQUENCY_CASE, COLLECTION_ACTION, PAYMENT_PROMISE, LITIGATION_CASE, RECOVERY, WRITE_OFF |
| ORG | Organization & control | ORG_UNIT, APP_USER, ROLE, USER_ROLE, DOCUMENT, AUDIT_LOG |

### 4.2 Overall entity relationships

The overall model is shown in two parts. Part A runs from the customer to the approved limit and its collateral. Part B runs from the limit through the contract and loan to post-loan management. `CREDIT_LIMIT` is the link between them. Each arrow is labelled with its cardinality (for example `1 : 0..n` means one to zero-or-many) and the relationship. Attributes are shown in the subject-area diagrams that follow.

**Part A: customer, application, approval, limit and collateral**

```mermaid
flowchart TB
    classDef ent fill:#eef2ff,stroke:#6366f1,color:#111
    GRP["CUSTOMER_GROUP"]:::ent
    GM["GROUP_MEMBER"]:::ent
    CUS["CUSTOMER"]:::ent
    RP["RELATED_PARTY"]:::ent
    FS["FINANCIAL_STATEMENT"]:::ent
    RAT["RATING_RECORD"]:::ent
    APP["CREDIT_APPLICATION"]:::ent
    FR["FACILITY_REQUEST"]:::ent
    TSK["APPROVAL_TASK"]:::ent
    OP["APPROVAL_OPINION"]:::ent
    DEC["APPROVAL_DECISION"]:::ent
    CND["APPROVAL_CONDITION"]:::ent
    LIM["CREDIT_LIMIT"]:::ent
    LTX["LIMIT_TXN"]:::ent
    LNK["COLLATERAL_LINK"]:::ent
    COL["COLLATERAL"]:::ent
    VAL["COLLATERAL_VALUATION"]:::ent
    GRP -->|"1 : 0..n · has"| GM
    CUS -->|"1 : 0..n · joins"| GM
    CUS -->|"1 : 0..n · links"| RP
    CUS -->|"1 : 0..n · reports"| FS
    CUS -->|"1 : 0..n · rated by"| RAT
    CUS -->|"1 : 0..n · submits"| APP
    APP -->|"1 : 1..n · contains"| FR
    APP -->|"1 : 0..n · routed to"| TSK
    TSK -->|"1 : 0..1 · produces"| OP
    APP -->|"1 : 0..1 · ends with"| DEC
    DEC -->|"1 : 0..n · imposes"| CND
    DEC -->|"1 : 0..n · creates"| LIM
    LIM -->|"1 : 0..n · records"| LTX
    LIM -->|"1 : 0..n · secured by"| LNK
    COL -->|"1 : 0..n · allocated via"| LNK
    COL -->|"1 : 0..n · valued by"| VAL
```

**Part B: limit, contract, loan and post-loan management**

```mermaid
flowchart TB
    classDef ent fill:#eef2ff,stroke:#6366f1,color:#111
    LIM["CREDIT_LIMIT"]:::ent
    CON["LOAN_CONTRACT"]:::ent
    SEC["SECURITY_CONTRACT"]:::ent
    COV["COVENANT"]:::ent
    LN["LOAN"]:::ent
    RS["REPAYMENT_SCHEDULE"]:::ent
    LTX["LOAN_TXN"]:::ent
    RC["RISK_CLASSIFICATION"]:::ent
    ECL["ECL_RESULT"]:::ent
    WS["WARNING_SIGNAL"]:::ent
    DC["DELINQUENCY_CASE"]:::ent
    CA["COLLECTION_ACTION"]:::ent
    PP["PAYMENT_PROMISE"]:::ent
    LIM -->|"1 : 0..n · funds"| CON
    CON -->|"1 : 0..n · backed by"| SEC
    CON -->|"1 : 0..n · includes"| COV
    CON -->|"1 : 0..n · drawn as"| LN
    LN -->|"1 : 1..n · follows"| RS
    LN -->|"1 : 0..n · posts"| LTX
    LN -->|"1 : 0..n · classified"| RC
    LN -->|"1 : 0..n · provisioned"| ECL
    LN -->|"1 : 0..n · raises"| WS
    LN -->|"1 : 0..n · overdue in"| DC
    DC -->|"1 : 0..n · logs"| CA
    DC -->|"1 : 0..n · records"| PP
```

Also in the model but omitted above for readability: the limit tree is self-referencing (`CREDIT_LIMIT.parent_limit_id`); `WARNING_SIGNAL`, `MONITORING_TASK` and `DELINQUENCY_CASE` are also keyed by `customer_id`; and `CUSTOMER` owns `COLLATERAL` through `owner_customer_id`.

### 4.3 Party

```mermaid
flowchart TB
    classDef ent fill:#eef2ff,stroke:#6366f1,color:#111
    CUS["<b>CUSTOMER</b><br/>customer_id (PK)<br/>customer_type<br/>id_type, id_number<br/>customer_name<br/>industry_code<br/>rm_user_id (FK)<br/>branch_id (FK)<br/>status"]:::ent
    ORG["<b>ORGANIZATION</b><br/>customer_id (PK, FK)<br/>established_date<br/>registered_capital<br/>legal_rep_name<br/>enterprise_size"]:::ent
    IND["<b>INDIVIDUAL</b><br/>customer_id (PK, FK)<br/>birth_date<br/>nationality<br/>employer_name<br/>monthly_income"]:::ent
    GRP["<b>CUSTOMER_GROUP</b><br/>group_id (PK)<br/>group_name<br/>parent_customer_id (FK)"]:::ent
    GM["<b>GROUP_MEMBER</b><br/>group_id (PK, FK)<br/>customer_id (PK, FK)<br/>ownership_pct<br/>control_type"]:::ent
    FS["<b>FINANCIAL_STATEMENT</b><br/>statement_id (PK)<br/>customer_id (FK)<br/>period_end<br/>statement_type<br/>audit_status"]:::ent
    CUS -->|"1 : 0..1 · is a"| ORG
    CUS -->|"1 : 0..1 · is a"| IND
    GRP -->|"1 : 0..n · has"| GM
    CUS -->|"1 : 0..n · joins"| GM
    CUS -->|"1 : 0..n · files"| FS
```

**CUSTOMER**

| Field | Type | Req | Description |
| --- | --- | --- | --- |
| customer_id | varchar(20) | Y | Bank-wide customer number (CIF), primary key |
| customer_type | varchar(10) | Y | IND (individual), ORG (organization), FI (financial institution) |
| id_type | varchar(10) | Y | Passport, national ID, corporate registration number, etc. |
| id_number | varchar(40) | Y | Unique together with id_type (BR-CUS-001) |
| customer_name | varchar(200) | Y | Legal name |
| customer_name_local | varchar(200) | N | Name in local script (e.g. kanji) |
| industry_code | varchar(10) | C | Required for ORG; national industry classification |
| country_code | char(2) | Y | ISO 3166 |
| region_code | varchar(10) | N | For concentration reporting |
| segment | varchar(10) | Y | LARGE, SME, MICRO, RETAIL, FI |
| rm_user_id | varchar(20) | Y | Responsible RM; drives data scope |
| branch_id | varchar(10) | Y | Owning branch |
| kyc_status | varchar(10) | Y | PENDING, CLEARED, EXPIRED, REJECTED |
| kyc_expiry_date | date | N | Next KYC refresh date |
| aml_risk_level | varchar(10) | Y | LOW, MEDIUM, HIGH |
| current_rating_grade | varchar(5) | N | Snapshot of latest effective rating |
| current_rating_date | date | N | |
| is_related_party | boolean | Y | Related to the bank (insiders, shareholders) |
| status | varchar(10) | Y | PROSPECT, ACTIVE, FROZEN, CLOSED |
| created_at / created_by | timestamp / varchar | Y | Audit columns, on every table |
| updated_at / updated_by | timestamp / varchar | Y | |
| version_no | integer | Y | Optimistic locking, on every table |

**GROUP_MEMBER**

| Field | Type | Req | Description |
| --- | --- | --- | --- |
| group_id | varchar(20) | Y | PK part |
| customer_id | varchar(20) | Y | PK part |
| ownership_pct | decimal(5,2) | N | Direct or indirect ownership |
| control_type | varchar(20) | Y | EQUITY, DE_FACTO, GUARANTEE_CHAIN, COMMON_MANAGEMENT |
| effective_from / effective_to | date | Y / N | Membership history |

**FINANCIAL_STATEMENT** and **FIN_STMT_ITEM**

| Field | Type | Req | Description |
| --- | --- | --- | --- |
| statement_id | varchar(20) | Y | PK |
| customer_id | varchar(20) | Y | FK |
| period_end | date | Y | Reporting date |
| period_type | varchar(5) | Y | FY, H1, Q1–Q4 |
| statement_type | varchar(10) | Y | CONSOLIDATED, STANDALONE |
| audit_status | varchar(10) | Y | AUDITED, REVIEWED, MANAGEMENT |
| currency | char(3) | Y | ISO 4217 |
| item_code (item table) | varchar(20) | Y | Standard chart (total assets, revenue, EBITDA…) |
| item_value (item table) | decimal(20,2) | Y | |

### 4.4 Application and approval

```mermaid
flowchart LR
    classDef ent fill:#eef2ff,stroke:#6366f1,color:#111
    APP["<b>CREDIT_APPLICATION</b><br/>application_id (PK)<br/>customer_id (FK)<br/>application_type<br/>total_requested_amount<br/>status<br/>required_approval_level"]:::ent
    FR["<b>FACILITY_REQUEST</b><br/>facility_req_id (PK)<br/>application_id (FK)<br/>product_code (FK)<br/>requested_amount<br/>tenor_months"]:::ent
    TSK["<b>APPROVAL_TASK</b><br/>task_id (PK)<br/>application_id (FK)<br/>step_no<br/>assignee_user_id (FK)<br/>task_status"]:::ent
    OP["<b>APPROVAL_OPINION</b><br/>opinion_id (PK)<br/>task_id (FK)<br/>opinion_type<br/>comments"]:::ent
    DEC["<b>APPROVAL_DECISION</b><br/>decision_id (PK)<br/>application_id (FK)<br/>decision_type<br/>valid_until"]:::ent
    CND["<b>APPROVAL_CONDITION</b><br/>condition_id (PK)<br/>decision_id (FK)<br/>condition_type<br/>condition_status"]:::ent
    APP -->|"1 : 1..n · contains"| FR
    APP -->|"1 : 0..n · routed to"| TSK
    TSK -->|"1 : 0..1 · produces"| OP
    APP -->|"1 : 0..1 · ends with"| DEC
    DEC -->|"1 : 0..n · imposes"| CND
```

**CREDIT_APPLICATION**

| Field | Type | Req | Description |
| --- | --- | --- | --- |
| application_id | varchar(20) | Y | PK |
| customer_id | varchar(20) | Y | FK → CUSTOMER |
| application_type | varchar(10) | Y | NEW, RENEWAL, INCREASE, CHANGE, ONEOFF |
| channel | varchar(10) | Y | BRANCH, RM, ONLINE, API |
| total_requested_amount | decimal(20,2) | Y | Sum of facility requests in reporting currency |
| reporting_currency | char(3) | Y | |
| purpose | varchar(500) | Y | |
| repayment_source | varchar(500) | Y | |
| rating_record_id | varchar(20) | C | Required before submit for ORG customers |
| proposed_pricing_summary | varchar(500) | N | |
| policy_exceptions | varchar(500) | N | Rule hits from BR checks |
| required_approval_level | varchar(10) | C | Set by authority routing at submit |
| status | varchar(15) | Y | See state machine 5.1 |
| submitted_by / submitted_at | varchar / timestamp | C | Set on submit |
| decision_id | varchar(20) | N | FK → APPROVAL_DECISION |

**FACILITY_REQUEST**

| Field | Type | Req | Description |
| --- | --- | --- | --- |
| facility_req_id | varchar(20) | Y | PK |
| application_id | varchar(20) | Y | FK |
| product_code | varchar(10) | Y | FK → PRODUCT |
| requested_amount | decimal(20,2) | Y | |
| currency | char(3) | Y | |
| tenor_months | integer | Y | |
| revolving_flag | boolean | Y | |
| rate_type | varchar(10) | Y | FIXED, FLOATING |
| base_rate_code | varchar(10) | C | Required if FLOATING |
| margin_bps | integer | Y | Spread in basis points |
| repayment_method | varchar(10) | Y | ANNUITY, EQUAL_PRIN, BULLET, BALLOON, CUSTOM |
| proposed_collateral_ids | array | N | References to COLLATERAL |
| approved_amount | decimal(20,2) | N | Filled by decision |

**APPROVAL_TASK / APPROVAL_OPINION**

| Field | Type | Req | Description |
| --- | --- | --- | --- |
| task_id | varchar(20) | Y | PK |
| application_id | varchar(20) | Y | FK |
| step_no | integer | Y | Order in the route |
| step_type | varchar(10) | Y | REVIEW, APPROVE, COMMITTEE |
| assignee_user_id | varchar(20) | Y | |
| assignee_role | varchar(20) | Y | |
| authority_level | varchar(10) | Y | |
| delegated_from | varchar(20) | N | If acting under delegation |
| task_status | varchar(10) | Y | OPEN, CLAIMED, DONE, RETURNED, CANCELLED |
| due_at | timestamp | Y | SLA |
| opinion_type (opinion) | varchar(15) | Y | AGREE, AGREE_COND, DISAGREE, RETURN, ESCALATE |
| comments (opinion) | text | Y | |
| conditions_proposed (opinion) | text | N | |
| acted_at (opinion) | timestamp | Y | |

**APPROVAL_DECISION / APPROVAL_CONDITION**

| Field | Type | Req | Description |
| --- | --- | --- | --- |
| decision_id | varchar(20) | Y | PK |
| application_id | varchar(20) | Y | FK |
| decision_type | varchar(15) | Y | APPROVED, APPROVED_COND, AMENDED, REJECTED |
| decided_by | varchar(20) | Y | Final approver or committee ID |
| decided_at | timestamp | Y | |
| valid_until | date | C | Required if approved (BR-APP-007) |
| condition_type (condition) | varchar(10) | Y | PRE_CONTRACT, PRE_DISB, POST_DISB |
| condition_text (condition) | varchar(1000) | Y | |
| due_date (condition) | date | C | Required for POST_DISB |
| condition_status (condition) | varchar(10) | Y | OPEN, MET, WAIVED, BREACHED |
| evidence_doc_id (condition) | varchar(20) | N | FK → DOCUMENT |

### 4.5 Rating

| Entity | Field | Type | Req | Description |
| --- | --- | --- | --- | --- |
| RATING_MODEL | model_id, model_version | varchar | Y | PK; versioned |
| | segment | varchar(10) | Y | Segment the model applies to |
| | effective_from / effective_to | date | Y / N | |
| RATING_RECORD | rating_id | varchar(20) | Y | PK |
| | customer_id | varchar(20) | Y | FK |
| | model_id, model_version | varchar | Y | Model used |
| | rating_date | date | Y | |
| | financial_statement_id | varchar(20) | C | Required for ORG |
| | model_score | decimal(8,4) | Y | |
| | model_grade | varchar(5) | Y | Grade before override |
| | override_grade | varchar(5) | N | |
| | override_reason | varchar(500) | C | Required if override |
| | final_grade | varchar(5) | Y | |
| | pd | decimal(9,6) | Y | Probability of default for the grade |
| | status | varchar(10) | Y | DRAFT, APPROVED, SUPERSEDED |
| | approved_by / approved_at | varchar / timestamp | C | |
| RATING_FACTOR_SCORE | rating_id, factor_code | varchar | Y | PK |
| | factor_value, factor_score | decimal | Y | Input and score per factor |

### 4.6 Limit

```mermaid
flowchart LR
    classDef ent fill:#eef2ff,stroke:#6366f1,color:#111
    LIM["<b>CREDIT_LIMIT</b><br/>limit_id (PK)<br/>parent_limit_id (FK)<br/>customer_id (FK)<br/>limit_level<br/>approved_amount<br/>used_amount<br/>status"]:::ent
    LTX["<b>LIMIT_TXN</b><br/>limit_txn_id (PK)<br/>limit_id (FK)<br/>txn_type<br/>amount<br/>source_ref"]:::ent
    LF["<b>LIMIT_FREEZE</b><br/>freeze_id (PK)<br/>limit_id (FK)<br/>reason_code<br/>freeze_status"]:::ent
    LIM -->|"1 : 0..n · parent of"| LIM
    LIM -->|"1 : 0..n · records"| LTX
    LIM -->|"1 : 0..n · frozen by"| LF
```

**CREDIT_LIMIT**

| Field | Type | Req | Description |
| --- | --- | --- | --- |
| limit_id | varchar(20) | Y | PK |
| parent_limit_id | varchar(20) | N | Null for the top node |
| limit_level | varchar(10) | Y | GROUP, CUSTOMER, FACILITY, SUBLIMIT |
| group_id | varchar(20) | C | Required for GROUP level |
| customer_id | varchar(20) | C | Required below GROUP |
| product_scope | array | C | Products allowed (FACILITY, SUBLIMIT) |
| currency | char(3) | Y | Limit currency |
| approved_amount | decimal(20,2) | Y | |
| used_amount | decimal(20,2) | Y | Derived from LIMIT_TXN; reconciled daily |
| available_amount | decimal(20,2) | Y | approved − used (never negative except approved overdraw) |
| revolving_flag | boolean | Y | |
| start_date / end_date | date | Y | |
| last_drawdown_date | date | N | No new occupation after this date |
| source_decision_id | varchar(20) | Y | FK → APPROVAL_DECISION |
| status | varchar(10) | Y | See state machine 5.2 |

**LIMIT_TXN** (append-only)

| Field | Type | Req | Description |
| --- | --- | --- | --- |
| limit_txn_id | varchar(30) | Y | PK |
| limit_id | varchar(20) | Y | FK |
| txn_type | varchar(10) | Y | OCCUPY, RELEASE, REVERSE, ADJUST |
| amount | decimal(20,2) | Y | In limit currency |
| original_amount / original_currency | decimal / char(3) | Y | Before conversion |
| fx_rate | decimal(18,8) | Y | Rate used |
| ccf | decimal(5,4) | Y | Credit conversion factor (1.0 for loans) |
| source_type | varchar(10) | Y | LOAN, GUARANTEE, LC, BILL, MANUAL |
| source_ref | varchar(30) | Y | ID of the source object |
| business_date | date | Y | |
| reversal_of | varchar(30) | N | Links a REVERSE to the original |

### 4.7 Collateral

```mermaid
flowchart LR
    classDef ent fill:#eef2ff,stroke:#6366f1,color:#111
    COL["<b>COLLATERAL</b><br/>collateral_id (PK)<br/>collateral_type<br/>owner_customer_id (FK)<br/>market_value<br/>eligible_value<br/>status"]:::ent
    VAL["<b>COLLATERAL_VALUATION</b><br/>valuation_id (PK)<br/>collateral_id (FK)<br/>valuation_date<br/>market_value<br/>valuer_type"]:::ent
    LNK["<b>COLLATERAL_LINK</b><br/>link_id (PK)<br/>collateral_id (FK)<br/>limit_id (FK)<br/>allocated_value<br/>lien_rank"]:::ent
    REG["<b>COLLATERAL_REGISTRATION</b><br/>registration_id (PK)<br/>collateral_id (FK)<br/>registry_no<br/>reg_status"]:::ent
    INS["<b>INSURANCE_POLICY</b><br/>policy_id (PK)<br/>collateral_id (FK)<br/>expiry_date<br/>bank_loss_payee"]:::ent
    COL -->|"1 : 0..n · valued by"| VAL
    COL -->|"1 : 0..n · allocated via"| LNK
    COL -->|"1 : 0..n · registered as"| REG
    COL -->|"1 : 0..n · insured by"| INS
```

**COLLATERAL**

| Field | Type | Req | Description |
| --- | --- | --- | --- |
| collateral_id | varchar(20) | Y | PK |
| collateral_type | varchar(10) | Y | RE_RES, RE_COM, LAND, DEPOSIT, BOND, SHARE, RECEIV, EQUIP, VEHICLE, OTHER |
| collateral_class | varchar(10) | Y | MORTGAGE, PLEDGE, LIEN |
| owner_customer_id | varchar(20) | Y | Owner, may be a third party |
| description | varchar(1000) | Y | |
| location / address | varchar(500) | C | Required for real estate |
| registry_reference | varchar(50) | C | Land registry, securities account, etc. |
| currency | char(3) | Y | |
| market_value | decimal(20,2) | Y | Snapshot of latest valuation |
| valuation_date | date | Y | |
| haircut_pct | decimal(5,2) | Y | From haircut table at valuation date |
| eligible_value | decimal(20,2) | Y | market_value × (1 − haircut) |
| allocated_total | decimal(20,2) | Y | Sum of active links (BR-COL-003) |
| next_revaluation_date | date | Y | BR-COL-005 |
| status | varchar(10) | Y | DRAFT, REGISTERED, PLEDGED, RELEASED, ENFORCED |

**COLLATERAL_LINK**

| Field | Type | Req | Description |
| --- | --- | --- | --- |
| link_id | varchar(20) | Y | PK |
| collateral_id | varchar(20) | Y | FK |
| limit_id | varchar(20) | C | Link at limit level, or |
| contract_id | varchar(20) | C | link at contract level (one of the two) |
| allocated_value | decimal(20,2) | Y | |
| lien_rank | integer | Y | 1 = first lien |
| effective_from / effective_to | date | Y / N | |
| link_status | varchar(10) | Y | ACTIVE, RELEASED |

**GUARANTEE** (personal or corporate guarantee as a credit mitigant)

| Field | Type | Req | Description |
| --- | --- | --- | --- |
| guarantee_id | varchar(20) | Y | PK |
| guarantor_customer_id | varchar(20) | Y | FK → CUSTOMER |
| guarantee_type | varchar(10) | Y | JOINT_SEVERAL, LIMITED, GUAR_COMPANY |
| guaranteed_amount | decimal(20,2) | Y | |
| limit_id / contract_id | varchar(20) | C | What it secures |
| start_date / end_date | date | Y | |

### 4.8 Contract and loan

```mermaid
flowchart LR
    classDef ent fill:#eef2ff,stroke:#6366f1,color:#111
    CON["<b>LOAN_CONTRACT</b><br/>contract_id (PK)<br/>limit_id (FK)<br/>customer_id (FK)<br/>product_code (FK)<br/>contract_amount<br/>maturity_date<br/>status"]:::ent
    LN["<b>LOAN</b><br/>loan_id (PK)<br/>contract_id (FK)<br/>disbursed_amount<br/>outstanding_principal<br/>dpd<br/>loan_status"]:::ent
    RS["<b>REPAYMENT_SCHEDULE</b><br/>loan_id (PK, FK)<br/>installment_no (PK)<br/>due_date<br/>principal_due<br/>interest_due<br/>inst_status"]:::ent
    LTX["<b>LOAN_TXN</b><br/>loan_txn_id (PK)<br/>loan_id (FK)<br/>txn_type<br/>amount<br/>value_date"]:::ent
    CON -->|"1 : 0..n · drawn as"| LN
    LN -->|"1 : 1..n · follows"| RS
    LN -->|"1 : 0..n · posts"| LTX
```

**LOAN_CONTRACT**

| Field | Type | Req | Description |
| --- | --- | --- | --- |
| contract_id | varchar(20) | Y | PK |
| contract_no | varchar(30) | Y | Printed contract number |
| customer_id | varchar(20) | Y | FK |
| limit_id | varchar(20) | Y | FK → CREDIT_LIMIT (facility level) |
| decision_id | varchar(20) | Y | FK → APPROVAL_DECISION |
| product_code | varchar(10) | Y | FK |
| contract_amount | decimal(20,2) | Y | ≤ approved facility amount (BR-CON-001) |
| currency | char(3) | Y | |
| rate_type / base_rate_code / margin_bps | varchar / varchar / integer | Y / C / Y | |
| repayment_method | varchar(10) | Y | |
| start_date / maturity_date | date | Y | |
| availability_end_date | date | Y | Last date to draw |
| template_id / template_version | varchar | Y | |
| has_deviation | boolean | Y | Non-standard clauses present |
| signed_date | date | C | |
| status | varchar(10) | Y | DRAFT, SIGNED, EFFECTIVE, TERMINATED, CLOSED |

**LOAN** (drawdown)

| Field | Type | Req | Description |
| --- | --- | --- | --- |
| loan_id | varchar(20) | Y | PK |
| contract_id | varchar(20) | Y | FK |
| core_account_no | varchar(30) | Y | Account in core banking |
| disbursed_amount | decimal(20,2) | Y | |
| disbursement_date | date | Y | |
| maturity_date | date | Y | |
| current_rate | decimal(9,6) | Y | All-in annual rate |
| outstanding_principal | decimal(20,2) | Y | |
| accrued_interest | decimal(20,2) | Y | |
| overdue_principal / overdue_interest | decimal(20,2) | Y | |
| penalty_interest | decimal(20,2) | Y | |
| dpd | integer | Y | Days past due of the oldest unpaid installment |
| next_due_date | date | N | |
| entrusted_payment_flag | boolean | Y | |
| restructured_flag | boolean | Y | |
| classification / ifrs9_stage | varchar(10) / smallint | Y | Snapshots of latest result |
| loan_status | varchar(10) | Y | See state machine 5.3 |

**REPAYMENT_SCHEDULE**

| Field | Type | Req | Description |
| --- | --- | --- | --- |
| loan_id + installment_no | varchar + integer | Y | PK |
| due_date | date | Y | |
| principal_due / interest_due / fee_due | decimal(20,2) | Y | |
| principal_paid / interest_paid / fee_paid | decimal(20,2) | Y | |
| inst_status | varchar(10) | Y | FUTURE, DUE, PAID, PARTIAL, OVERDUE |
| schedule_version | integer | Y | Incremented on reschedule; old versions kept |

**LOAN_TXN** (append-only)

| Field | Type | Req | Description |
| --- | --- | --- | --- |
| loan_txn_id | varchar(30) | Y | PK |
| loan_id | varchar(20) | Y | FK |
| txn_type | varchar(10) | Y | DISBURSE, REPAY, PREPAY, ACCRUE, PENALTY, FEE, WRITEOFF, RECOVER, REVERSE |
| amount | decimal(20,2) | Y | |
| principal_part / interest_part / penalty_part / fee_part | decimal(20,2) | Y | Allocation (BR-SRV-001) |
| value_date / business_date | date | Y | |
| core_txn_ref | varchar(40) | Y | Link to core banking posting |

### 4.9 Monitoring and asset quality

| Entity | Field | Type | Req | Description |
| --- | --- | --- | --- | --- |
| WARNING_SIGNAL | signal_id | varchar(20) | Y | PK |
| | signal_code | varchar(10) | Y | From signal catalog (EW-01…) |
| | customer_id / loan_id | varchar(20) | Y / N | |
| | source | varchar(10) | Y | INTERNAL, BUREAU, COURT, REGISTRY, NEWS, MANUAL |
| | severity | varchar(10) | Y | RED, ORANGE, YELLOW |
| | detected_at | timestamp | Y | |
| | assigned_to | varchar(20) | Y | |
| | due_at | timestamp | Y | By severity |
| | action_code | varchar(10) | N | NONE, WATCH, FREEZE, REDUCE, COLLATERAL, DOWNGRADE, RECALL |
| | signal_status | varchar(10) | Y | OPEN, IN_PROGRESS, CLOSED |
| MONITORING_TASK | task_id | varchar(20) | Y | PK |
| | task_type | varchar(10) | Y | FIRST_CHECK, PERIODIC, RENEWAL, SIGNAL |
| | customer_id | varchar(20) | Y | |
| | due_date | date | Y | |
| | report_id | varchar(20) | N | FK → MONITORING_REPORT when done |
| COVENANT | covenant_id | varchar(20) | Y | PK |
| | contract_id | varchar(20) | Y | FK |
| | covenant_type | varchar(10) | Y | FINANCIAL, INFORMATION, NEGATIVE |
| | metric_code / operator / threshold | varchar / varchar / decimal | C | e.g. DSCR ≥ 1.2 (financial covenants) |
| | test_frequency | varchar(10) | Y | QUARTERLY, SEMIANNUAL, ANNUAL |
| COVENANT_TEST | test_id, covenant_id, test_date | — | Y | |
| | actual_value, result | decimal / varchar | Y | PASS, BREACH, WAIVED |
| RISK_CLASSIFICATION | loan_id + as_of_date | varchar + date | Y | PK; one row per period |
| | system_class | varchar(10) | Y | NORMAL, SPECIAL, SUBSTD, DOUBTFUL, LOSS |
| | final_class | varchar(10) | Y | |
| | ifrs9_stage | smallint | Y | 1, 2, 3 |
| | adjust_reason | varchar(500) | C | Required if final ≠ system |
| | approved_by | varchar(20) | C | |
| ECL_RESULT | loan_id + as_of_date + scenario | — | Y | PK |
| | pd_12m / pd_lifetime | decimal(9,6) | Y | |
| | lgd | decimal(9,6) | Y | |
| | ead | decimal(20,2) | Y | |
| | ecl_amount | decimal(20,2) | Y | Scenario-weighted total on the weighted row |
| | method | varchar(10) | Y | COLLECTIVE, INDIVIDUAL |

### 4.10 Collection

| Entity | Field | Type | Req | Description |
| --- | --- | --- | --- | --- |
| DELINQUENCY_CASE | case_id | varchar(20) | Y | PK |
| | customer_id | varchar(20) | Y | One open case per customer |
| | opened_date | date | Y | |
| | dpd_bucket | varchar(10) | Y | B1 (1–30), B2 (31–60), B3 (61–90), B4 (90+) |
| | total_overdue | decimal(20,2) | Y | |
| | strategy_code | varchar(10) | Y | |
| | collector_id / agency_id | varchar(20) | Y / N | |
| | case_status | varchar(10) | Y | OPEN, PROMISED, LEGAL, RESTRUCT, RECOVERED, WRITTEN_OFF, CLOSED |
| COLLECTION_ACTION | action_id, case_id | varchar | Y | |
| | action_type | varchar(10) | Y | CALL, SMS, LETTER, VISIT, LEGAL_NOTICE |
| | outcome_code | varchar(10) | Y | CONTACTED, NO_ANSWER, PROMISE, REFUSED, DISPUTE |
| | action_at / action_by | timestamp / varchar | Y | |
| PAYMENT_PROMISE | promise_id, case_id | varchar | Y | |
| | promised_date / promised_amount | date / decimal | Y | |
| | promise_status | varchar(10) | Y | PENDING, KEPT, BROKEN |
| WRITE_OFF | writeoff_id, loan_id | varchar | Y | |
| | writeoff_amount / writeoff_date | decimal / date | Y | |
| | approval_ref | varchar(20) | Y | |
| | recovered_after | decimal(20,2) | Y | Running total of post-write-off recoveries |

### 4.11 Organization and control

| Entity | Purpose | Key fields |
| --- | --- | --- |
| ORG_UNIT | Branches and departments in a tree | org_id, parent_org_id, org_type, org_name |
| APP_USER | System users | user_id, org_id, employee_no, status |
| ROLE / USER_ROLE | Roles and assignment | role_id; user_id + role_id + data_scope (OWN, BRANCH, REGION, BANK) + valid dates |
| AUTHORITY_RULE | Authority matrix rows | rule_id, role_id, product_group, segment, max_amount, min_rating, collateral_condition, approval_level, version |
| AUTHORITY_DELEGATION | Temporary delegation | from_user, to_user, scope, valid_from, valid_to, approved_by |
| DOCUMENT | Files attached to any object | doc_id, owner_type, owner_id, doc_type, storage_ref, hash, uploaded_by |
| AUDIT_LOG | Immutable change log | log_id, object_type, object_id, action, before_json, after_json, user_id, ip, timestamp |

### 4.12 Code tables (reference data)

| Code table | Values (examples) |
| --- | --- |
| CUSTOMER_TYPE | IND, ORG, FI |
| SEGMENT | LARGE, SME, MICRO, RETAIL, FI |
| APPLICATION_TYPE | NEW, RENEWAL, INCREASE, CHANGE, ONEOFF |
| LIMIT_LEVEL | GROUP, CUSTOMER, FACILITY, SUBLIMIT |
| COLLATERAL_TYPE | RE_RES, RE_COM, LAND, DEPOSIT, BOND, SHARE, RECEIV, EQUIP, VEHICLE, OTHER |
| REPAYMENT_METHOD | ANNUITY, EQUAL_PRIN, BULLET, BALLOON, CUSTOM |
| CLASSIFICATION | NORMAL, SPECIAL, SUBSTD, DOUBTFUL, LOSS |
| DPD_BUCKET | B0 (current), B1 (1–30), B2 (31–60), B3 (61–90), B4 (90+) |
| SIGNAL_SEVERITY | RED, ORANGE, YELLOW |
| CONDITION_TYPE | PRE_CONTRACT, PRE_DISB, POST_DISB |

### 4.13 Data design rules

| # | Rule |
| --- | --- |
| DR-1 | Master entities keep a current snapshot; history lives in dated child tables (rating, valuation, classification, schedule versions) |
| DR-2 | Limit balances are derived from the append-only LIMIT_TXN ledger and reconciled daily; direct updates to used_amount are not allowed |
| DR-3 | Loan balances are mastered in core banking; the CMS copy is reconciled daily and breaks are raised as exceptions |
| DR-4 | Every amount has a currency; cross-currency aggregates store the FX rate and date used |
| DR-5 | Ratings, classifications and limits are bitemporal: effective date plus record timestamp, to answer "what did we know on date X" |
| DR-6 | Personal data (ID numbers, income, contact details) is encrypted at rest and masked by role |
| DR-7 | No physical delete of business records; status changes only, with audit log |
| DR-8 | Every table carries created/updated audit columns and a version number for optimistic locking |

---

## 5. State machines

### 5.1 Credit application

```mermaid
flowchart TB
    classDef st fill:#eef2ff,stroke:#6366f1,color:#111
    classDef term fill:#f3f4f6,stroke:#9ca3af,color:#111
    START((start)):::term
    DRAFT["DRAFT"]:::st
    SUBMITTED["SUBMITTED"]:::st
    IN_REVIEW["IN_REVIEW"]:::st
    RETURNED["RETURNED"]:::st
    IN_APPROVAL["IN_APPROVAL"]:::st
    IN_COMMITTEE["IN_COMMITTEE"]:::st
    APPROVED["APPROVED"]:::st
    ACTIVATED["ACTIVATED"]:::term
    REJECTED["REJECTED"]:::term
    EXPIRED["EXPIRED"]:::term
    WITHDRAWN["WITHDRAWN"]:::term
    START -->|"RM creates"| DRAFT
    DRAFT -->|"RM submits"| SUBMITTED
    DRAFT -->|"RM cancels"| WITHDRAWN
    SUBMITTED -->|"analyst claims task"| IN_REVIEW
    IN_REVIEW -->|"incomplete"| RETURNED
    RETURNED -->|"RM resubmits"| SUBMITTED
    RETURNED -->|"RM cancels"| WITHDRAWN
    IN_REVIEW -->|"review opinion given"| IN_APPROVAL
    IN_APPROVAL -->|"above authority"| IN_COMMITTEE
    IN_APPROVAL -->|"approver approves"| APPROVED
    IN_APPROVAL -->|"approver rejects"| REJECTED
    IN_COMMITTEE -->|"vote passes"| APPROVED
    IN_COMMITTEE -->|"vote fails"| REJECTED
    APPROVED -->|"limit activated"| ACTIVATED
    APPROVED -->|"valid_until passed"| EXPIRED
```

### 5.2 Credit limit

```mermaid
flowchart LR
    classDef st fill:#eef2ff,stroke:#6366f1,color:#111
    classDef term fill:#f3f4f6,stroke:#9ca3af,color:#111
    START((start)):::term
    PENDING["PENDING"]:::st
    ACTIVE["ACTIVE"]:::st
    FROZEN["FROZEN"]:::st
    EXPIRED["EXPIRED"]:::st
    CLOSED["CLOSED"]:::term
    START -->|"approval decision"| PENDING
    PENDING -->|"contract effective, conditions met"| ACTIVE
    PENDING -->|"approval validity passed"| EXPIRED
    ACTIVE -->|"warning signal or manual freeze"| FROZEN
    FROZEN -->|"unfreeze approved"| ACTIVE
    ACTIVE -->|"end_date reached"| EXPIRED
    ACTIVE -->|"cancelled"| CLOSED
    FROZEN -->|"cancelled"| CLOSED
    EXPIRED -->|"all exposures settled"| CLOSED
```

Allowed operations by status:

| Status | Occupy | Release | Adjust | Freeze |
| --- | --- | --- | --- | --- |
| PENDING | No | No | Yes | No |
| ACTIVE | Yes | Yes | Yes | Yes |
| FROZEN | No | Yes | Yes | — |
| EXPIRED | No | Yes | No | No |
| CLOSED | No | No | No | No |

### 5.3 Loan (drawdown)

```mermaid
flowchart TB
    classDef st fill:#eef2ff,stroke:#6366f1,color:#111
    classDef term fill:#f3f4f6,stroke:#9ca3af,color:#111
    START((start)):::term
    PENDING_DISB["PENDING_DISB"]:::st
    NORMAL["NORMAL"]:::st
    OVERDUE["OVERDUE"]:::st
    RESTRUCTURED["RESTRUCTURED"]:::st
    NPL["NPL"]:::st
    CLOSED["CLOSED"]:::term
    WRITTEN_OFF["WRITTEN_OFF"]:::term
    SOLD["SOLD"]:::term
    CANCELLED["CANCELLED"]:::term
    START -->|"drawdown approved"| PENDING_DISB
    PENDING_DISB -->|"disbursed"| NORMAL
    PENDING_DISB -->|"posting failed or cancelled"| CANCELLED
    NORMAL -->|"installment unpaid"| OVERDUE
    OVERDUE -->|"overdue cleared"| NORMAL
    OVERDUE -->|"DPD over 90 or Substandard and worse"| NPL
    NORMAL -->|"terms eased"| RESTRUCTURED
    OVERDUE -->|"terms eased"| RESTRUCTURED
    NPL -->|"restructuring approved"| RESTRUCTURED
    RESTRUCTURED -->|"probation passed"| NORMAL
    RESTRUCTURED -->|"probation failed"| NPL
    NPL -->|"write-off approved"| WRITTEN_OFF
    NPL -->|"NPL sale"| SOLD
    NORMAL -->|"fully repaid"| CLOSED
    NPL -->|"fully recovered"| CLOSED
```

### 5.4 Collateral

```mermaid
flowchart LR
    classDef st fill:#eef2ff,stroke:#6366f1,color:#111
    classDef term fill:#f3f4f6,stroke:#9ca3af,color:#111
    START((start)):::term
    DRAFT["DRAFT"]:::st
    VALUED["VALUED"]:::st
    PLEDGED["PLEDGED"]:::st
    RELEASED["RELEASED"]:::term
    ENFORCED["ENFORCED"]:::term
    START -->|"registered in CMS"| DRAFT
    DRAFT -->|"valuation approved"| VALUED
    VALUED -->|"linked and legally registered"| PLEDGED
    PLEDGED -->|"revaluation"| PLEDGED
    PLEDGED -->|"no secured exposure, release approved"| RELEASED
    PLEDGED -->|"enforcement completed"| ENFORCED
```

---

## 6. Business events

Events are published after the business transaction commits. Consumers include other CMS domains, the data warehouse, notification services and core banking adapters.

| Event | Published by | Payload (key fields) | Main consumers |
| --- | --- | --- | --- |
| E-CUSTOMER-ACTIVATED | Party | customer_id, segment, rm_user_id | Origination, DWH |
| E-RATING-APPROVED | Rating | customer_id, rating_id, final_grade, pd | Approval, Pricing, Asset quality |
| E-APP-SUBMITTED | Origination | application_id, customer_id, amount | Approval (routing), notifications |
| E-APP-DECIDED | Approval | application_id, decision_id, decision_type | Limit, notifications, DWH |
| E-LIMIT-ACTIVATED | Limit | limit_id, customer_id, amount | Contract, DWH |
| E-LIMIT-FROZEN | Limit | limit_id, reason_code | Disbursement, notifications |
| E-COLLATERAL-REVALUED | Collateral | collateral_id, eligible_value, ltv | Monitoring (LTV check), ECL |
| E-CONTRACT-EFFECTIVE | Contract | contract_id, limit_id | Limit activation, disbursement |
| E-LOAN-DISBURSED | Loan | loan_id, amount, value_date | Limit (occupy), GL, DWH, monitoring (first check) |
| E-LOAN-REPAID | Loan | loan_id, allocation | Limit (release), collection |
| E-LOAN-OVERDUE | Loan | loan_id, dpd, overdue_amount | Collection, monitoring (EW-01/02) |
| E-SIGNAL-RAISED | Monitoring | signal_id, severity, customer_id | RM tasks; Limit (auto-freeze on RED) |
| E-CLASS-CHANGED | Asset quality | loan_id, old_class, new_class, stage | GL, reporting, collection |
| E-ECL-POSTED | Asset quality | period, totals | GL, reporting |
| E-LOAN-WRITTEN-OFF | Collection | loan_id, amount | GL, limit (close), reporting |
| E-LOAN-CLOSED | Loan | loan_id | Limit, collateral release check |

---

## 7. Business rules catalog

| Rule ID | Area | Rule | Enforcement |
| --- | --- | --- | --- |
| BR-CUS-001 | Customer | ID type + ID number is unique across all customers | Hard block |
| BR-CUS-002 | Customer | A sanctions or PEP hit blocks onboarding until Compliance clears it | Hard block |
| BR-CUS-003 | Customer | Ownership ≥ 50% or de facto control places the customer in the same group | Proposed by system, confirmed by RM |
| BR-CUS-004 | Customer | Financial statement: total assets = total liabilities + equity (tolerance 0.5%) | Hard block |
| BR-APP-001 | Application | One open application per customer per facility type | Hard block |
| BR-APP-005 | Approval | A user acts at most once per application and never on an application they created | Hard block |
| BR-APP-007 | Approval | Approval expires if the limit is not activated within 180 days (configurable) | Batch |
| BR-RAT-002 | Rating | Override ≤ 2 notches needs risk approval; > 2 notches needs CRO | Routing |
| BR-RAT-003 | Rating | Rating older than 12 months cannot support a new approval | Hard block |
| BR-AUT-001 | Authority | Approval level = highest level required by amount, rating, product, collateral and policy exceptions | Routing |
| BR-AUT-002 | Authority | Group exposure (existing + requested) is used for the amount test, not the single application | Routing |
| BR-PRC-001 | Pricing | Price below the grid floor for the rating is a pricing exception and raises approval level by one | Routing |
| BR-LIM-001 | Limit | Child limits cannot exceed their parent; sum of children may exceed the parent only if marked "shared" | Hard block |
| BR-LIM-003 | Limit | Occupation passes availability, status, product-scope and concentration checks at every level, atomically | Hard block |
| BR-LIM-004 | Limit | RED warning signal freezes all active limits of the customer automatically | Automatic |
| BR-COL-001 | Collateral | Real estate above the threshold amount needs a panel valuer | Hard block |
| BR-COL-003 | Collateral | Total allocations ≤ eligible value | Hard block |
| BR-COL-005 | Collateral | Revaluation overdue raises a warning signal (EW-08 family) | Batch |
| BR-COL-007 | Collateral | Collateral is released only when no active link to an exposure remains | Hard block |
| BR-CON-001 | Contract | Contract terms must be within the approved terms (amount ≤, tenor ≤, price ≥, collateral ⊇) | Hard block |
| BR-DIS-001 | Disbursement | Maker ≠ checker | Hard block |
| BR-DIS-002 | Disbursement | Entrusted payment is mandatory above the product threshold | Hard block |
| BR-DIS-003 | Disbursement | If core banking posting fails, the limit occupation is reversed | Automatic |
| BR-SRV-001 | Servicing | Repayment allocation order: fees → penalty → overdue interest → overdue principal → current interest → current principal | Automatic |
| BR-CHG-001 | Loan change | Easing terms for a borrower with DPD > 0 or class ≥ Special mention is flagged as restructuring | Automatic flag |
| BR-CLS-001 | Classification | DPD-based floors apply: 91+ DPD is at least Substandard | Automatic |
| BR-CLS-002 | Classification | All loans of one customer take the worst classification (configurable for retail) | Automatic |
| BR-CLS-003 | Classification | An upgrade needs one approval level higher than a downgrade | Routing |
| BR-WOF-001 | Write-off | Written-off accounts remain tracked off-balance-sheet for recovery | Automatic |

---

## 8. Functional module design (microservices)

This section breaks the system into deployable functional modules, one microservice (or small group of services) each. It refines the capabilities in section 2.3 to function level, using a reference functional scope for a full-lifecycle credit system: product factory, customer management, credit application and approval, contract and disbursement, loan management, limit control, credit accounting, post-loan management, guarantee management, risk warning, asset preservation, archives, reports, process configuration and a unified portal.

### 8.1 Module map

```mermaid
flowchart TB
    classDef layer fill:#eef2ff,stroke:#6366f1,color:#111
    classDef shared fill:#ecfdf5,stroke:#10b981,color:#111
    classDef ext fill:#f3f4f6,stroke:#9ca3af,color:#111
    ACCESS["<b>Access layer</b><br/>M01 Unified Portal · API gateway / channel access"]:::layer
    PRE["<b>Pre-loan services</b><br/>M02 Product Center · M03 Customer Management<br/>M04 Rating Center · M05 Credit Application & Approval"]:::layer
    IN["<b>In-loan services</b><br/>M06 Contract & Disbursement · M07 Limit Control · M08 Collateral Center<br/>M09 Guarantee Management · M10 Loan Management · M11 Credit Accounting"]:::layer
    POST["<b>Post-loan services</b><br/>M12 Post-loan Center · M14 Asset Preservation"]:::layer
    SHARED["<b>Shared business services</b><br/>M13 Risk Control · M15 Archives Center<br/>M16 Report Center · M17 Process Center"]:::shared
    PLAT["<b>Technical platform</b><br/>Rule engine & model service · Message bus · Users, roles & data scope<br/>Notification · File storage · Batch scheduler"]:::ext
    EXT["<b>Core & external systems</b><br/>Core banking · General ledger · Credit bureau · External data"]:::ext
    ACCESS --> PRE --> IN --> POST
    PRE -.uses.-> SHARED
    IN -.uses.-> SHARED
    POST -.uses.-> SHARED
    SHARED --> PLAT
    IN <-->|posting, balances| EXT
    PRE <-->|bureau, registry data| EXT
```

### 8.2 Module catalog

| ID | Module | Maps to capability | Owns data (section 4 / 10) | Main consumers |
| --- | --- | --- | --- | --- |
| M01 | Unified Portal | C-18 | User workbench, preferences, notices | All users |
| M02 | Product Center | C-02, C-08 | PRODUCT, PRODUCT_VERSION, PRODUCT_COMPONENT | M05, M06, M07, M10, M11 |
| M03 | Customer Management | C-01 | CUSTOMER and party entities | All modules |
| M04 | Rating Center | C-04 | RATING_MODEL, RATING_RECORD | M05, M07, M13, M12 |
| M05 | Credit Application & Approval | C-03, C-05, C-09 | CREDIT_APPLICATION, APPROVAL_* | M07, M06 |
| M06 | Contract & Disbursement | C-10, C-11 | LOAN_CONTRACT, DISBURSEMENT_REQUEST | M10, M07, M11 |
| M07 | Limit Control | C-06 | CREDIT_LIMIT, LIMIT_TXN, LIMIT_FREEZE | M05, M06, M13 |
| M08 | Collateral Center | C-07 | COLLATERAL and related entities | M05, M06, M12, M14 |
| M09 | Guarantee Management | C-07 | GUARANTEE, GUARANTOR_EXPOSURE, GUARANTEE_COMPANY | M05, M06, M14 |
| M10 | Loan Management | C-12 | LOAN, REPAYMENT_SCHEDULE, LOAN_TXN | M11, M12, M14 |
| M11 | Credit Accounting | C-12, C-14 | ACCOUNTING_EVENT, ACCOUNTING_ENTRY | GL, M17 reports |
| M12 | Post-loan Center | C-13 | MONITORING_*, FUND_FLOW_CHECK, SCREENING_CAMPAIGN | M13, M14 |
| M13 | Risk Control | C-13, C-14 | RISK_RULE, RISK_HIT, WARNING_SIGNAL, RISK_CLASSIFICATION | All business modules |
| M14 | Asset Preservation | C-15 | NPL_ASSET, DISPOSAL_PLAN, LITIGATION_CASE, WRITE_OFF | M11, reporting |
| M15 | Archives Center | C-18 | ARCHIVE_FILE, ARCHIVE_ITEM, ARCHIVE_BORROW | Audit, all modules |
| M16 | Report Center | C-17 | REPORT_TEMPLATE, REPORT_INSTANCE | M05, M12, management |
| M17 | Process Center | C-18 | PROCESS_DEFINITION, PROCESS_INSTANCE, TASK | All workflow modules |

**Service boundaries:** each module owns its tables and exposes them only through APIs and events (AP-6). Cross-module reads that need speed (for example, customer name on a task list) use cached read models fed by events, not direct database joins.

---

### 8.3 M01 Unified Portal

One entry point for all credit staff. It hosts the screens of every module and gives each user a single place to find their work.

| Function | Description | Key rules |
| --- | --- | --- |
| Single sign-on | One login across all modules, linked to the bank's directory | Session timeout; MFA for approvers |
| Workbench | To-do, in-progress, completed and returned tasks across all processes | Sorted by SLA due time; overdue tasks highlighted |
| Notifications | System messages, warning signals, approval results, expiry reminders | Delivered in-app, plus email/SMS by notice type |
| Customer 360° view | Customer profile, group, rating, limits, exposures, collateral, loans, signals, files on one page | Data masked by role (DR-6) |
| Global search | Search customers, applications, contracts, loans, collateral by number or name | Results filtered by data scope |
| Personal dashboard | Role-specific KPIs: pipeline, approvals pending, overdue, warning signals | Built on Report Center (M16) |
| Delegation | Users set temporary delegation of tasks (with approval) | BR-AUT delegation rules |
| Channel access | Exposes services to internet banking, mobile and partner APIs through the API gateway | Channel identity and rate limits |

---

### 8.4 M02 Product Center (product factory)

Products are **assembled from reusable components** instead of being coded, so a new product can be launched by configuration.

```mermaid
flowchart TB
    classDef cms fill:#eef2ff,stroke:#6366f1,color:#111
    classDef side fill:#f3f4f6,stroke:#9ca3af,color:#111
    COMP["<b>Reusable components</b><br/>Basic info · Customer eligibility · Amount & term · Rate & pricing<br/>Repayment methods · Collateral & guarantee requirements · Fees<br/>Limit occupation rules · Process binding · Document checklist<br/>Report templates · Accounting rules"]:::side
    PV["<b>Product version</b><br/>components assembled, effective date"]:::cms
    COMP --> PV
    PV --> D1[Draft]:::cms
    D1 -->|submit| D2[Under review]:::cms
    D2 -->|approved| D3[Listed]:::cms
    D2 -->|rejected| D1
    D3 -->|suspend| D4[Suspended]:::cms
    D4 -->|resume| D3
    D3 -->|delist| D5[Delisted]:::side
    D4 -->|delist| D5
```

| Function | Description | Key rules |
| --- | --- | --- |
| Product catalog | Tree of product categories: corporate loans, retail loans, trade finance, guarantees, bills, interbank | Each product has one category |
| Product configuration | Fill each component for a product version | Mandatory components by category |
| Component library | Maintain reusable components shared by many products | A change to a shared component creates new product versions, not silent edits |
| Versioning | Every change creates a new version with an effective date | Existing contracts keep the version they were signed under |
| Listing / delisting | Draft → review → listed → suspended → delisted | Listing needs Risk + Product approval (maker-checker); delisting stops new applications only |
| Channel availability | Which channels can sell the product | |
| Product test | Simulate an application against the product rules before listing | Required before first listing |

**Product component detail**

| Component | Example parameters |
| --- | --- |
| Customer eligibility | Customer types, segments, industries allowed / excluded, minimum rating, age range (retail) |
| Amount & term | Min / max amount, min / max tenor, grace period |
| Rate & pricing | Fixed / floating, base rate, margin range by rating, penalty rate multiple |
| Repayment methods | Allowed methods, repayment frequency, first repayment rule |
| Collateral & guarantee | Required mitigant types, maximum LTV, guarantor requirements |
| Fees | Arrangement, commitment, prepayment fee rules |
| Limit occupation | Revolving or not, credit conversion factor, which limit type it uses |
| Process binding | Which approval and disbursement processes apply (from M17) |
| Document checklist | Required documents per process step |
| Report templates | Investigation and approval report templates (from M16) |
| Accounting rules | GL account mapping per accounting event (from M11) |

---

### 8.5 M03 Customer Management

Covers four customer types with a common core and type-specific extensions.

| Customer type | Specific data | Specific functions |
| --- | --- | --- |
| Corporate | Registration, shareholders, management, financial statements, related enterprises | Financial statement import and spreading; shareholder tree |
| Individual | Identity, employment, income, household, assets | Income verification; household / spouse linkage |
| Peer (financial institution) | Licence type, regulator, capital adequacy, external ratings | Interbank limit (counterparty limit) under M07 |
| Group | Member list, parent company, control relationships | Group identification, group limit, group exposure view |

| Function | Description | Key rules |
| --- | --- | --- |
| Create / modify / query | Maintain customer records with maker-checker on key fields | BR-CUS-001 uniqueness; changes to key fields (name, ID, industry) need review |
| Delete (logical) | Close a customer with no active business | Not allowed while any limit, contract or loan is active (DR-7) |
| Customer merge | Merge duplicate records | Moves all business to the surviving ID; audit trail kept |
| Group identification | System proposes group links from shareholding and guarantee chains | BR-CUS-003; confirmed by RM, approved by risk |
| Related party management | Bank insiders and related enterprises | Related-party lending limits checked by M13 |
| Financial statements | Import, spread to standard chart, ratio analysis | BR-CUS-004 balance check |
| External data | Credit bureau, registry, court, tax data pulled on demand or on schedule | Consent recorded |
| Customer status | Prospect, active, frozen, closed; blacklist and watch list flags | Blacklisted customers blocked at application (M13) |
| Customer transfer | Reassign RM or branch | Approval; open tasks move with the customer |

---

### 8.6 M04 Rating Center

| Function | Description | Key rules |
| --- | --- | --- |
| Model management | Maintain rating models and scorecards per segment, with versions | Model change needs validation sign-off |
| Corporate rating | Quantitative (financials) + qualitative factors → score → grade → PD | P-04 |
| Individual rating / scoring | Application scorecard and behavior scorecard | Behavior score refreshed monthly |
| Group rating | Group-level rating from parent and consolidated data; member rating capped by group rating unless stand-alone strength is proven | Ceiling rule configurable |
| Peer rating | Financial institution model (capital, asset quality, liquidity, external rating) | |
| Facility rating | LGD by collateral type and seniority | Feeds ECL and pricing |
| Override and approval | Override with reason and authority | BR-RAT-002 |
| Re-rating | Annual, and on trigger events from M13 | BR-RAT-003 |
| Rating history and migration | Grade history, migration matrices | Input to model monitoring |

---

### 8.7 M05 Credit Application & Approval

Covers all corporate and individual credit across the bank, so every branch follows the same standardized process and approval can be centralized.

| Function | Description | Key rules |
| --- | --- | --- |
| Credit application | New, renewal, increase, change, one-off; multiple facilities per application | BR-APP-001 |
| Comprehensive credit line | One application grants a customer-level line split into facility limits | Limit tree created in M07 |
| Group credit | One application for a group with member allocations | Group limit ≥ sum of member allocations unless shared |
| Low-risk business | Fast track for fully cash-secured or bank-guaranteed business | Simplified route; still occupies limits |
| Due diligence | Investigation report built from template (M16), documents attached | Mandatory sections by product |
| Pricing proposal | Price checked against the pricing grid | BR-PRC-001 |
| Risk pre-check | M13 rules run at submit: blacklist, industry, concentration, related party | Hard rules block; soft rules warn |
| Approval routing | Route from authority matrix and process definition (M17) | BR-AUT-001, BR-AUT-002 |
| Review and approval | Independent review, approval, countersign, committee | BR-APP-005 |
| Approval report | Generated from template with review opinions (M16) | Locked after decision |
| Reconsideration | Customer appeal of a rejection; new review by a higher level | One appeal per application |
| Centralized approval | Applications from all branches routed to a central approval center by product and amount | Workload balancing by queue |

---

### 8.8 M06 Contract & Disbursement

Manages the request and approval of each **use of credit** (drawdown, guarantee issue, L/C issue), balancing customer access to funds with risk control.

| Function | Description | Key rules |
| --- | --- | --- |
| Contract generation | From template and approval; security and guarantee contracts generated alongside | BR-CON-001 |
| Contract signing | Paper or e-signature; signing record | Signatory authority check |
| Disbursement application | Customer or RM requests a drawdown under a contract | Within contract availability period |
| Disbursement approval | Credit Admin maker-checker with checklist; higher-risk items routed to approvers | BR-DIS-001 |
| Condition verification | Check approval conditions and document checklist | All PRE_DISB conditions MET or WAIVED |
| Entrusted payment | Pay suppliers directly from the loan | BR-DIS-002; beneficiary checked against purpose |
| Self-directed payment | Borrower pays; use of funds verified after the fact by M12 | Evidence due within N days |
| Off-balance issuance | Issue guarantees, L/Cs, acceptance bills under the limit | Occupies limit at CCF |
| Core banking posting | Create loan account and post disbursement | BR-DIS-003 rollback |
| Contract changes | Amendments, extensions, termination | Re-approval where terms change (P-10) |

---

### 8.9 M07 Limit Control

Unified management of the **creation, use and release** of credit limits for all asset businesses across all business lines (corporate, retail, interbank, trade, investment).

| Function | Description | Key rules |
| --- | --- | --- |
| Limit setup | Limit tree created from approval | BR-LIM-001 |
| Limit types | Customer comprehensive limit; single facility limit; group limit; interbank (peer) counterparty limit; industry / country / product portfolio limits | Each limit type has its own occupation rules |
| Real-time occupation | Synchronous check-and-occupy API for every module and channel | BR-LIM-003, all levels atomic |
| Pre-occupation | Reserve limit at approval of a disbursement, confirm on posting, release on cancel | Pre-occupations expire after N days |
| Release | On repayment (revolving), maturity, cancellation | Non-revolving never re-opens |
| Limit adjustment | Increase, reduce, extend, transfer between members of a group | Via approval |
| Limit allocation | Split a group or customer limit to members, branches or channels | Sum ≤ parent unless shared |
| Freeze / unfreeze | Manual or automatic (M13 RED signal) | BR-LIM-004 |
| Dynamic monitoring | Utilization, approaching expiry, overdrawn, concentration dashboards | Alerts at 80% and 100% utilization |
| Reconciliation | Rebuild used amount from LIMIT_TXN and exposures daily | DR-2 |

---

### 8.10 M08 Collateral Center

Full-lifecycle management of collateral information, valuation and registration. Detailed in P-06 and section 4.7.

| Function | Description | Key rules |
| --- | --- | --- |
| Collateral registration | Record collateral and owner, with type-specific attribute sets (real estate, deposit, securities, receivables, equipment, vehicles, IP) | Attribute template per type |
| Valuation | Internal, external (panel), and automated (index-based) valuation | BR-COL-001 |
| Revaluation | Scheduled and event-driven | BR-COL-005 |
| Registration management | Mortgage / pledge registration with registry, certificate numbers, expiry | Registration before disbursement |
| Warehouse (vault) management | Original title documents in and out of the vault | Dual control |
| Insurance | Policies, expiry alerts, claims | Bank as loss payee |
| Allocation | Allocate value across limits / contracts with lien rank | BR-COL-003 |
| Monitoring | LTV, value drop, expiry signals to M13 | |
| Release and enforcement | Release after repayment; hand over to M14 for enforcement | BR-COL-007 |

---

### 8.11 M09 Guarantee Management

Manages guarantees **received** as credit mitigation (personal, corporate, guarantee-company guarantees). Bank-**issued** guarantees (L/Gs) are products in M02 that occupy limits in M07.

| Function | Description | Key rules |
| --- | --- | --- |
| Guarantor registration | Guarantor is a customer in M03 | KYC applies |
| Guarantee contract | Type (joint and several, limited), amount, period, scope | Linked to limit or contract |
| Guarantor capacity | Track each guarantor's total guaranteed amount across the bank | Capacity check against guarantor's financial strength |
| Mutual / chain guarantee detection | Detect A guarantees B and B guarantees A, and longer chains | Raises risk signal (M13); counts toward group identification |
| Guarantee company management | Cooperation agreements, margin deposits, compensation limits, performance tracking | Total exposure per company capped |
| Guarantee claims | Call on guarantor or guarantee company when borrower defaults | Hand-off to M14 |

---

### 8.12 M10 Loan Management

| Function | Description | Key rules |
| --- | --- | --- |
| Loan account view | Balances, schedule, transactions, rate history | Mirror of core banking, reconciled daily (DR-3) |
| Repayment | Scheduled, early, partial, full; allocation order | BR-SRV-001 |
| Rate management | Floating rate reset, rate change approval | |
| Extension / rescheduling | Change requests with approval | BR-CHG-001 |
| Loan transfer | Transfer between branches or to asset preservation | |
| Five-category / stage display | Current classification and history | From M13 |
| Interest and fee calculation | Accrual, penalty, compound interest where allowed | Day-count per product |

---

### 8.13 M11 Credit Accounting

Turns credit business events into accounting entries for the general ledger, using rules configured per product (M02).

```mermaid
flowchart LR
    E[Business event<br/>from M06, M10, M13, M14] --> R[Accounting rule lookup<br/>product + event type]
    R --> J[Generate entries<br/>debit / credit]
    J --> V{Balanced?}
    V -- no --> X[Exception queue]
    V -- yes --> P[Post to GL interface]
    P --> RC[Daily reconciliation<br/>sub-ledger vs GL]
```

**Accounting events and entries (example)**

| Event | Debit | Credit |
| --- | --- | --- |
| Loan disbursed | Loans and advances | Customer deposit / clearing |
| Interest accrued | Interest receivable | Interest income |
| Repayment received | Customer deposit / clearing | Loans and advances; interest receivable |
| Loan becomes overdue | Overdue loans | Loans and advances (normal) |
| Penalty interest accrued | Interest receivable (penalty) | Interest income (penalty), or off-balance memo if non-accrual |
| Moved to non-accrual (NPL) | Off-balance interest memo | — (stop accruing on balance sheet) |
| ECL provision increased | Impairment loss | Loan loss allowance |
| Loan written off | Loan loss allowance | Loans and advances |
| Recovery after write-off | Customer deposit / clearing | Other income / allowance |
| Guarantee / L/C issued (off-balance) | Off-balance commitment | Off-balance contra |
| Fee received | Customer deposit | Fee income (or deferred income for amortized fees) |

| Function | Description | Key rules |
| --- | --- | --- |
| Accounting rule configuration | Event × product → entry template with GL accounts | Rule change needs Finance approval |
| Sub-ledger | Credit sub-ledger by loan / contract | Balanced per event |
| Non-accrual handling | Stop on-balance interest at NPL; track in memo | Per local rule (e.g. 90 DPD) |
| Provision posting | Post ECL from M13 month-end | E-ECL-POSTED |
| Reconciliation | Sub-ledger vs GL vs core banking | Breaks to exception queue |
| Accounting reports | Trial balance, interest income, provision movement | Via M16 |

---

### 8.14 M12 Post-loan Center

Three functional groups: **fund monitoring**, **post-loan inspection** and **risk screening**.

| Group | Function | Description | Key rules |
| --- | --- | --- | --- |
| Fund monitoring | Use-of-funds tracking | Follow disbursed funds through the borrower's accounts | First check within 15–30 days |
| | Abnormal flow detection | Funds returned to the borrower or related parties, moved to restricted uses (e.g. securities, real estate speculation), split into many small transfers | Rule-based (M13); hits create signals |
| | Repayment source monitoring | Account inflows vs expected repayment | EW-03 |
| Post-loan inspection | First inspection | Confirm funds used as stated | P-11 |
| | Periodic inspection | Scheduled by risk level; on-site or off-site; inspection report from M16 template | Frequency table in 3.12 |
| | Special inspection | Triggered by signals or management request | Due by severity |
| | Covenant tests | Financial and information covenants | COVENANT_TEST |
| | Collateral inspection | Site check of collateral condition | Photo evidence |
| Risk screening | Screening campaign | Bulk review of a portfolio slice by criteria (industry, region, product, rating, exposure size), e.g. after a sector shock | SCREENING_CAMPAIGN |
| | Task distribution | Campaign creates review tasks per customer for RMs | Deadline per campaign |
| | Result consolidation | Findings, actions, re-classification, summary report | Feeds M13 and management |

---

### 8.15 M13 Risk Control

Identifies and **intercepts** risk across the whole business flow, using configurable rules and models.

```mermaid
flowchart LR
    A[Customer onboarding] --> B[Credit application]
    B --> C[Approval]
    C --> D[Contract]
    D --> E[Disbursement]
    E --> F[Post-loan]
    F --> G[Collection / preservation]
    R1([Blacklist, sanctions,<br/>AML]) -.-> A
    R2([Policy rules: industry,<br/>concentration, related party,<br/>large exposure]) -.-> B
    R3([Authority, pricing,<br/>exceptions]) -.-> C
    R4([Terms within approval]) -.-> D
    R5([Limit, freeze, conditions,<br/>fund purpose]) -.-> E
    R6([Warning rules & models,<br/>fund flow, covenants]) -.-> F
    R7([Classification, ECL]) -.-> G
```

| Function | Description | Key rules |
| --- | --- | --- |
| Rule management | Define rules: condition, interception point, action (block, warn, route up, notify), effective dates | Rule change needs Risk approval; versioned |
| Model management | Early-warning and fraud models, scores and thresholds | Model monitoring monthly |
| Interception | Rules called synchronously at each interception point | Response < 300 ms for online flows |
| Warning signal management | Signal catalog, grading, assignment, handling, closure | P-11 |
| Blacklist / watch list | Bank-wide lists with reasons and expiry | Blacklist hit blocks new credit |
| Risk classification | Month-end classification and adjustments | P-12 |
| Concentration control | Industry, region, product, single-name and group caps | BR-LIM-003 |
| Rule hit log | Every evaluation and hit recorded with input data | Audit and rule tuning |

**Rule action types**

| Action | Effect |
| --- | --- |
| BLOCK | Stop the transaction; reason shown |
| WARN | Continue; warning visible to all later approvers |
| ESCALATE | Raise required approval level |
| REQUIRE | Add a mandatory document or condition |
| NOTIFY | Send notice to risk / compliance |
| SIGNAL | Create a warning signal |

---

### 8.16 M14 Asset Preservation

Daily management and disposal of non-performing assets, to reduce losses and maximize recovery.

```mermaid
flowchart TD
    A[Loan classified NPL<br/>or transferred] --> B[NPL asset created<br/>owner: special assets team]
    B --> C[Asset assessment:<br/>borrower, guarantors, collateral,<br/>recovery estimate]
    C --> D[Disposal plan]
    D --> D1[Collection & negotiation]
    D --> D2[Restructuring]
    D --> D3[Litigation & enforcement]
    D --> D4[Debt-to-asset settlement]
    D --> D5[NPL transfer / sale]
    D --> D6[Write-off]
    D1 --> E[Recovery recorded]
    D2 --> E
    D3 --> E
    D4 --> F[Foreclosed asset management]
    D5 --> E
    D6 --> G[Off-balance tracking<br/>continued recovery]
    F --> E
    G --> E
```

| Function | Description | Key rules |
| --- | --- | --- |
| NPL transfer | Move loans to the preservation team with full file | Automatic at NPL or by decision |
| Asset ledger | One record per NPL asset: principal, interest, costs, collateral, guarantors | Daily balances from M10 |
| Disposal plan | Chosen route(s), expected recovery, timeline; approved by authority | BR per amount |
| Litigation management | Cases, courts, hearings, judgments, enforcement, legal costs | Statute of limitation reminders |
| Debt-to-asset (foreclosed assets) | Assets received in settlement: valuation, holding, sale | Holding period limits |
| NPL transfer / sale | Portfolio packaging, valuation, bidding, settlement | Board / committee approval |
| Write-off | Evidence, approval, off-balance tracking | BR-WOF-001 |
| Recovery accounting | Allocate recoveries to costs, interest, principal | Events to M11 |
| Performance | Recovery rate, time to recover, cost of recovery per team | Via M16 |

---

### 8.17 M15 Archives Center

Unified management of credit, disbursement and post-loan files (electronic and physical) to keep them complete and secure, and to support access, audit and risk tracing.

| Function | Description | Key rules |
| --- | --- | --- |
| File collection | Each business object (customer, application, contract, loan, collateral) has a file; documents are added as the process runs | Checklist per product (M02) |
| Completeness check | Archive officer checks required items before filing | Missing items returned to owner |
| Physical archive | Box, shelf and location for paper originals; barcode labels | Dual control for originals |
| Electronic archive | Scans and native e-documents with hash for integrity | Tamper detection by hash |
| Borrow and return | Request, approval, lending, due date, return | Overdue borrowing alerts |
| Access control | Access by role and data scope; every view logged | Audit trail |
| Retention and destruction | Retention period per file type; destruction needs approval | No destruction during litigation hold |
| Search | By customer, contract, loan, collateral, document type | |

---

### 8.18 M16 Report Center

Two kinds of reports: **business documents** used inside processes, and **management reports**.

| Type | Examples | Design |
| --- | --- | --- |
| Business documents | Investigation (due diligence) report, review report, approval report, post-loan inspection report, disbursement checklist, collection letters | Template with sections and fields; fields auto-filled from business data; free-text sections edited online; versioned; locked when the process step completes; exported to PDF |
| Management reports | Portfolio by product / industry / region / rating, approval pipeline, limit utilization, NPL and provision movement, post-loan task completion, collection performance | Built on data warehouse; parameters; scheduled distribution; drill-down |
| Regulatory reports | Large exposures, related-party lending, asset quality, interbank exposure | Per regulator format; data lineage to source |

| Function | Description | Key rules |
| --- | --- | --- |
| Template designer | Define sections, fields, data bindings, mandatory items | Template version bound to product (M02) |
| Online editing | Edit report within the task; auto-save | Only the task owner can edit |
| Display and viewing | Read-only view for approvers and auditors | Viewing logged |
| Report generation | PDF / Word output with watermark | Final version archived to M15 |
| Statistics | Ad-hoc query and dashboards for management | Data scope applies |

---

### 8.19 M17 Process Center

Unified configuration and management of all credit business processes, so processes can be standardized and adjusted without code changes.

| Function | Description | Key rules |
| --- | --- | --- |
| Process definition | Visual designer: nodes (task, approval, countersign, committee, automatic, sub-process), transitions, conditions | Versioned; running cases keep their version |
| Assignment rules | Who handles each node: role + org level + data scope, specific user, previous handler's manager, pool / queue | Segregation of duties (BR-APP-005) |
| Routing conditions | Expressions on business data (amount, rating, product, rule hits) | Can call authority matrix (C-09) |
| Countersign | Parallel review by several roles; all, majority or any | |
| Return and withdraw | Return to any earlier node; originator withdraw before first action | Reason mandatory |
| SLA and escalation | Due time per node, reminders, auto-escalation | Business calendar (holidays) |
| Process monitoring | Running instances, bottlenecks, average time per node | Feeds management dashboards |
| Process binding | Product × business type → process definition | Configured in M02 |

**Standard process catalog**

| Process | Used by |
| --- | --- |
| Customer onboarding / key-field change | M03 |
| Rating approval / override | M04 |
| Credit approval (by level: branch, regional, head office, committee) | M05 |
| Disbursement approval | M06 |
| Limit adjustment / freeze / unfreeze | M07 |
| Collateral registration / release | M08 |
| Loan change (extension, rescheduling) | M10 |
| Post-loan inspection | M12 |
| Classification adjustment | M13 |
| Disposal plan / write-off | M14 |
| File borrowing | M15 |
| Product listing / delisting | M02 |

---

## 9. Additional business processes

These processes come from the module design in section 8 and extend the process inventory in 3.1.

| ID | Process | Module | Trigger | Output |
| --- | --- | --- | --- | --- |
| P-16 | Product listing and delisting | M02 | New product or change | Listed / delisted product version |
| P-17 | Fund use monitoring | M12, M13 | Disbursement | Use-of-funds confirmation or signal |
| P-18 | Risk screening campaign | M12 | Management decision, sector event | Screening findings and actions |
| P-19 | Archive lifecycle | M15 | Business object created | Complete, filed archive |
| P-20 | Credit accounting and reconciliation | M11 | Every business event; end of day | GL postings; reconciliation report |
| P-21 | Interbank (peer) counterparty limit | M03, M04, M05, M07 | Treasury / interbank business need | Approved counterparty limit |

### 9.1 P-16 Product listing and delisting

| Step | Actor | Activity | Rules |
| --- | --- | --- | --- |
| 1 | Product manager | Create product version from components | Mandatory components by category |
| 2 | Product manager | Bind processes, report templates, accounting rules | Bound items must be in "active" status |
| 3 | Product manager | Run product test (simulated applications) | Test results attached |
| 4 | Risk | Review eligibility, pricing, collateral rules | Opinion recorded |
| 5 | Finance | Review accounting rules | Opinion recorded |
| 6 | Approver | Approve listing with effective date | Maker ≠ checker |
| 7 | System | Publish product version to channels | E-PRODUCT-LISTED |
| 8 | Product manager | Suspend or delist when needed | Delisting blocks new applications only; existing contracts unaffected |

### 9.2 P-17 Fund use monitoring

```mermaid
flowchart TD
    A[E-LOAN-DISBURSED] --> B{Payment type}
    B -- entrusted --> C[Beneficiary and amount<br/>match purchase contract?]
    B -- self-directed --> D[Request use-of-funds evidence<br/>within N days]
    C -- yes --> F[Monitor account flows<br/>for 90 days]
    C -- no --> S[Signal: purpose mismatch]
    D --> E{Evidence received<br/>and consistent?}
    E -- yes --> F
    E -- no --> S
    F --> G{Abnormal flow rules}
    G -- funds back to borrower<br/>or related party --> S
    G -- restricted use --> S
    G -- none --> H[Close monitoring:<br/>use confirmed]
    S --> I[Post-loan task to RM<br/>via M12]
```

Abnormal flow rules (examples, configured in M13):

| Rule | Description | Default action |
| --- | --- | --- |
| FF-01 | Over 30% of disbursed amount returns to the borrower or a related party within 30 days | SIGNAL (orange) |
| FF-02 | Funds transferred to securities, futures or crypto accounts | SIGNAL (red) |
| FF-03 | Funds used to repay other loans (unless that is the approved purpose) | SIGNAL (orange) |
| FF-04 | Payment split into many transfers just under the entrusted-payment threshold | SIGNAL (orange) |
| FF-05 | Beneficiary is on the blacklist or is a shell company | SIGNAL (red) |

### 9.3 P-18 Risk screening campaign

| Step | Actor | Activity |
| --- | --- | --- |
| 1 | Risk management | Define campaign: name, reason, criteria (industry, region, product, rating, exposure size), deadline |
| 2 | System | Select matching customers and loans; show count and exposure before launch |
| 3 | Risk management | Approve and launch |
| 4 | System | Create one screening task per customer for the responsible RM |
| 5 | RM | Complete screening checklist; record findings and proposed actions |
| 6 | Risk | Review findings; approve actions (watch list, freeze, downgrade, re-classification) |
| 7 | System | Execute approved actions in M07, M13 |
| 8 | Risk | Consolidated campaign report (M16) to management |

### 9.4 P-19 Archive lifecycle

```mermaid
flowchart LR
    A[Business object created] --> B[Archive file opened]
    B --> C[Documents added<br/>during the process]
    C --> D[Completeness check<br/>by archive officer]
    D -- missing --> C
    D -- complete --> E[Filed:<br/>e-archive + physical location]
    E --> F[Borrow / view<br/>with approval]
    F --> E
    E --> G[Business closed]
    G --> H[Retention period]
    H --> I[Destruction approved]
```

### 9.5 P-20 Credit accounting and reconciliation

| Time | Activity | Module |
| --- | --- | --- |
| Real time | Each business event creates an ACCOUNTING_EVENT; entries generated from the product's rules | M11 |
| Real time | Entries sent to the GL interface | M11 → GL |
| End of day | Accrual and overdue events from the daily batch | M10 → M11 |
| End of day | Reconcile credit sub-ledger vs GL by account and vs core banking by loan | M11 |
| Next morning | Breaks report; finance and operations resolve; corrections posted as new events | M11, M16 |
| Month end | ECL provision events from M13; non-accrual transfers | M13 → M11 |

### 9.6 P-21 Interbank (peer) counterparty limit

| Step | Activity | Notes |
| --- | --- | --- |
| 1 | Onboard financial institution as a peer customer | Licence, regulator, capital data |
| 2 | Peer rating | Financial institution model in M04 |
| 3 | Counterparty limit application by product line (money market, bond, FX, derivatives, trade finance) | One application, sub-limits per line |
| 4 | Approval by interbank credit authority | Separate authority matrix rows |
| 5 | Limit set up in M07 with sub-limits | Occupied by treasury systems through the real-time API |
| 6 | Daily monitoring of utilization and the institution's warning signals | Rating agency changes feed M13 |

---

## 10. Additional data entities

These entities support the modules in section 8. Common audit columns (created/updated by and at, version_no) apply to every table (DR-8).

### 10.1 Product

| Entity | Field | Type | Req | Description |
| --- | --- | --- | --- | --- |
| PRODUCT | product_code | varchar(10) | Y | PK |
| | product_name | varchar(100) | Y | |
| | category_code | varchar(10) | Y | Catalog node |
| | business_line | varchar(10) | Y | CORP, RETAIL, INTERBANK, TRADE |
| | current_version | integer | Y | Latest listed version |
| | status | varchar(10) | Y | DRAFT, REVIEW, LISTED, SUSPENDED, DELISTED |
| PRODUCT_VERSION | product_code + version_no | varchar + integer | Y | PK |
| | effective_from / effective_to | date | Y / N | |
| | approved_by / approved_at | varchar / timestamp | Y | |
| PRODUCT_COMPONENT | product_code + version_no + component_type | — | Y | PK |
| | component_ref | varchar(20) | C | Reference to a shared library component, or |
| | config_json | json | C | inline parameters (one of the two) |

### 10.2 Process

| Entity | Field | Type | Req | Description |
| --- | --- | --- | --- | --- |
| PROCESS_DEFINITION | process_key + version_no | varchar + integer | Y | PK |
| | process_name | varchar(100) | Y | |
| | definition_json | json | Y | Nodes, transitions, conditions |
| | status | varchar(10) | Y | DRAFT, ACTIVE, RETIRED |
| PROCESS_INSTANCE | instance_id | varchar(30) | Y | PK |
| | process_key + version_no | — | Y | Definition used |
| | business_type + business_id | varchar | Y | The application, loan, etc. |
| | current_node | varchar(30) | Y | |
| | started_at / ended_at | timestamp | Y / N | |
| | instance_status | varchar(10) | Y | RUNNING, COMPLETED, TERMINATED |
| TASK | task_id | varchar(30) | Y | PK (generalizes APPROVAL_TASK) |
| | instance_id, node_id | varchar | Y | |
| | assignee / candidate_group | varchar | C | User, or pool |
| | due_at | timestamp | Y | SLA |
| | task_status | varchar(10) | Y | OPEN, CLAIMED, DONE, RETURNED, CANCELLED |

### 10.3 Report and archive

| Entity | Field | Type | Req | Description |
| --- | --- | --- | --- | --- |
| REPORT_TEMPLATE | template_id + version_no | — | Y | PK |
| | report_type | varchar(20) | Y | INVESTIGATION, REVIEW, APPROVAL, INSPECTION, LETTER |
| | layout_json | json | Y | Sections, fields, data bindings |
| REPORT_INSTANCE | report_id | varchar(30) | Y | PK |
| | template_id + version_no | — | Y | |
| | business_type + business_id | varchar | Y | |
| | content_json | json | Y | Field values and edited text |
| | report_status | varchar(10) | Y | DRAFT, FINAL, LOCKED |
| | pdf_doc_id | varchar(20) | N | FK → DOCUMENT when generated |
| ARCHIVE_FILE | archive_id | varchar(20) | Y | PK |
| | business_type + business_id | varchar | Y | Owner object |
| | completeness_status | varchar(10) | Y | OPEN, INCOMPLETE, COMPLETE |
| | physical_location | varchar(50) | N | Room, shelf, box |
| | retention_until | date | N | Set at business closure |
| | legal_hold | boolean | Y | Blocks destruction |
| ARCHIVE_ITEM | item_id, archive_id | varchar | Y | |
| | doc_type | varchar(20) | Y | From checklist |
| | doc_id | varchar(20) | C | Electronic copy |
| | original_held | boolean | Y | Physical original in archive |
| ARCHIVE_BORROW | borrow_id, archive_id | varchar | Y | |
| | borrower_user_id, purpose | varchar | Y | |
| | approved_by | varchar(20) | Y | |
| | due_date / returned_date | date | Y / N | |

### 10.4 Risk control and post-loan

| Entity | Field | Type | Req | Description |
| --- | --- | --- | --- | --- |
| RISK_RULE | rule_id + version_no | — | Y | PK |
| | interception_point | varchar(20) | Y | ONBOARD, APPLY, APPROVE, CONTRACT, DISBURSE, POSTLOAN |
| | condition_expr | text | Y | Rule expression |
| | action_type | varchar(10) | Y | BLOCK, WARN, ESCALATE, REQUIRE, NOTIFY, SIGNAL |
| | severity | varchar(10) | C | For SIGNAL |
| | effective_from / effective_to | date | Y / N | |
| RISK_HIT | hit_id | varchar(30) | Y | PK |
| | rule_id + version_no | — | Y | |
| | business_type + business_id | varchar | Y | |
| | input_snapshot | json | Y | Data evaluated |
| | action_taken | varchar(10) | Y | |
| | hit_at | timestamp | Y | |
| FUND_FLOW_CHECK | check_id, loan_id | varchar | Y | |
| | payment_type | varchar(10) | Y | ENTRUSTED, SELF |
| | evidence_due_date | date | C | For SELF |
| | monitoring_until | date | Y | |
| | result | varchar(10) | Y | OPEN, CONFIRMED, ABNORMAL |
| SCREENING_CAMPAIGN | campaign_id | varchar(20) | Y | PK |
| | reason, criteria_json | text / json | Y | |
| | target_count, target_exposure | integer / decimal | Y | At launch |
| | deadline | date | Y | |
| | campaign_status | varchar(10) | Y | DRAFT, LAUNCHED, CLOSED |

### 10.5 Guarantee, accounting and asset preservation

| Entity | Field | Type | Req | Description |
| --- | --- | --- | --- | --- |
| GUARANTOR_EXPOSURE | guarantor_customer_id + as_of_date | — | Y | PK; daily snapshot |
| | total_guaranteed | decimal(20,2) | Y | Across all guarantees |
| | capacity_limit | decimal(20,2) | N | Approved capacity |
| GUARANTEE_COMPANY | company_customer_id | varchar(20) | Y | PK |
| | agreement_no, agreement_end | varchar / date | Y | |
| | margin_ratio | decimal(5,2) | Y | Margin deposit required |
| | max_exposure | decimal(20,2) | Y | Cap on bank exposure guaranteed |
| | compensation_rate | decimal(5,2) | N | Performance on past claims |
| ACCOUNTING_EVENT | event_id | varchar(30) | Y | PK |
| | event_type | varchar(20) | Y | DISBURSE, ACCRUE, REPAY, OVERDUE, NONACCRUAL, PROVISION, WRITEOFF, RECOVER, OFFBAL_ISSUE, FEE |
| | product_code + version_no | — | Y | Rule source |
| | source_type + source_id | varchar | Y | Loan, contract, etc. |
| | amount, currency, value_date | — | Y | |
| | post_status | varchar(10) | Y | PENDING, POSTED, FAILED |
| ACCOUNTING_ENTRY | entry_id, event_id | varchar | Y | |
| | gl_account | varchar(20) | Y | |
| | dr_cr | char(1) | Y | D or C |
| | amount | decimal(20,2) | Y | Sum of D = sum of C per event |
| | gl_ref | varchar(40) | N | GL posting reference |
| NPL_ASSET | npl_id | varchar(20) | Y | PK |
| | loan_id, customer_id | varchar | Y | |
| | transfer_date | date | Y | Into preservation |
| | principal, interest, costs | decimal(20,2) | Y | Claim amounts |
| | owner_user_id | varchar(20) | Y | Special assets officer |
| | npl_status | varchar(10) | Y | OPEN, IN_DISPOSAL, RECOVERED, SOLD, WRITTEN_OFF, CLOSED |
| DISPOSAL_PLAN | plan_id, npl_id | varchar | Y | |
| | route | varchar(15) | Y | COLLECT, RESTRUCT, LITIGATE, DEBT_ASSET, SALE, WRITEOFF |
| | expected_recovery | decimal(20,2) | Y | |
| | approved_by | varchar(20) | Y | |
| | plan_status | varchar(10) | Y | DRAFT, APPROVED, EXECUTING, DONE |
| FORECLOSED_ASSET | asset_id, npl_id | varchar | Y | Asset received in settlement |
| | book_value, valuation_date | decimal / date | Y | |
| | disposal_deadline | date | Y | Holding period limit |

### 10.6 Additional events and rules

| Event | Published by | Main consumers |
| --- | --- | --- |
| E-PRODUCT-LISTED / E-PRODUCT-DELISTED | M02 | Channels, M05 |
| E-RISK-HIT | M13 | Originating module, audit |
| E-FUNDFLOW-ABNORMAL | M12 | M13 (signal), RM task |
| E-CAMPAIGN-LAUNCHED | M12 | M01 (tasks) |
| E-NPL-TRANSFERRED | M14 | M11, M10, reporting |
| E-ACCOUNTING-POSTED / E-ACCOUNTING-FAILED | M11 | M16, operations |
| E-ARCHIVE-COMPLETE | M15 | Process close-out |

| Rule ID | Area | Rule |
| --- | --- | --- |
| BR-PRD-001 | Product | A product version cannot be listed without process, report-template and accounting-rule bindings |
| BR-PRD-002 | Product | Delisting blocks new applications; existing contracts keep their version |
| BR-GUA-001 | Guarantee | Total guaranteed by one guarantor ≤ approved guarantor capacity |
| BR-GUA-002 | Guarantee | Mutual or chain guarantees among applicants raise an orange signal and escalate approval |
| BR-ACC-001 | Accounting | Every accounting event must balance (sum debits = sum credits) before posting |
| BR-ACC-002 | Accounting | Interest stops accruing on-balance-sheet when a loan becomes non-accrual |
| BR-FF-001 | Fund monitoring | Self-directed payment evidence overdue by more than 10 days raises a yellow signal |
| BR-ARC-001 | Archives | A business object cannot be closed until its archive is COMPLETE |
| BR-ARC-002 | Archives | Files under legal hold cannot be destroyed |
| BR-NPL-001 | Asset preservation | A disposal plan with expected loss above the threshold needs committee approval |

---

## 11. Open questions

These items depend on the bank's jurisdiction, licence and policy, and must be confirmed before functional specification.

| # | Question | Affects |
| --- | --- | --- |
| Q-1 | Which regulator's classification scheme applies (e.g. FSA Japan, NFRA China), and is IFRS 9 or local GAAP used for provisioning? | P-12, RISK_CLASSIFICATION, ECL_RESULT |
| Q-2 | Which segments are in scope for phase 1: corporate, SME, retail? | All processes, rating models |
| Q-3 | Which core banking system is used, and does it or the CMS own the repayment schedule? | P-09, LOAN, REPAYMENT_SCHEDULE |
| Q-4 | Is the credit committee digital (voting in the system) or paper-based with minutes uploaded? | P-02 |
| Q-5 | Are guarantee companies (保証会社) used for retail products, and how are their approvals integrated? | P-03, GUARANTEE |
| Q-6 | Approval validity, revaluation frequencies, haircuts and DPD thresholds: confirm the bank's policy values | Parameters |
| Q-7 | Data residency and retention periods for credit files and personal data | DR-6, archiving |
| Q-8 | "Guarantee management": does it mean guarantees the bank receives (assumed here, M09), bank-issued guarantees (L/Gs), or both? | M09, M02 |
| Q-9 | Service granularity: deploy the 17 modules as separate microservices, or group some (e.g. M15–M17 as one shared service) for the first release? | Architecture, operations |
| Q-10 | Which accounting standard and local rules decide the non-accrual point and the GL account structure? | M11 |
| Q-11 | Is a separate treasury system the owner of interbank exposures, with the CMS holding only the counterparty limits? | P-21, M07 |
