# Credit Management System: Departments and Roles

As of Nov 16, 2026

A bank credit management system typically involves about **15–20 internal departments** and **around 40 user roles**, plus 8–10 types of external parties and several automated system actors. A small bank may merge many of these; a large bank may split them further, for example by separating corporate, SME and retail lines.

## 1. Front office: business departments

These departments originate credit and own the customer relationship.

| Department | Roles | What they do in the system |
| --- | --- | --- |
| Corporate Banking | Relationship Manager (RM), Assistant RM, Team Leader, Department Head | Onboard customers, create credit applications, write the investigation report, monitor accounts after the loan |
| SME / Small Business Banking | SME RM, SME Team Leader | Same as corporate, with simpler, more standardized products |
| Retail / Consumer Lending | Loan Officer, Branch Sales, Telesales / Online Sales | Take mortgage, auto and personal loan applications, mostly automated |
| Branch Network | Branch Manager, Branch Credit Officer, Teller / Counter Staff | Accept applications, collect documents, do first-level approval within branch authority |
| Trade Finance / Transaction Banking | Trade Finance Officer | Handle L/Cs, guarantees and supply-chain finance, all of which use credit limits |

## 2. Middle office: risk and credit

These departments independently assess, approve and monitor credit risk.

| Department | Roles | What they do in the system |
| --- | --- | --- |
| Credit Approval | Credit Analyst / Reviewer, Senior Approver, Chief Credit Officer (CCO) | Independently review applications, give opinions, approve within delegated authority |
| Credit Committee | Committee Secretary, Committee Members, Chairperson | Approve large or high-risk cases, with voting and minutes recorded in the system |
| Risk Management | Credit Risk Manager, Portfolio Risk Analyst, Chief Risk Officer (CRO) | Set policies, limits and concentration caps; run portfolio analysis and stress testing |
| Risk Modeling / Model Validation | Model Developer, Model Validator | Build and maintain rating and scorecard models, validate them independently, set parameters |
| Post-Loan Monitoring / Early Warning | Monitoring Officer, EWS Analyst | Handle early-warning signals, periodic reviews, the watch list and covenant checks |
| Asset Quality / Special Assets | Classification Officer, Workout Officer | Classify loans (IFRS 9 stages / five categories), restructure problem loans, plan recovery |

## 3. Back office: operations

These departments execute, record and service what has been approved.

| Department | Roles | What they do in the system |
| --- | --- | --- |
| Credit Administration / Loan Operations | Credit Admin Officer (maker), Credit Admin Supervisor (checker) | Check conditions before disbursement, set up contracts, enter and verify limits, approve disbursement |
| Collateral Management | Collateral Officer, Valuation Coordinator | Register collateral, track valuations and revaluations, manage mortgage registrations and insurance |
| Collections / Recovery | Collector (early stage), Senior Collector (late stage), Collection Manager | Manage overdue accounts, record payment promises, assign agencies |
| Legal | Legal Counsel, Litigation Officer | Review contract templates and non-standard clauses, handle lawsuits and enforcement |
| Document Center / Archives | Archive Officer | Manage original documents and e-archives, handle retrieval requests |

## 4. Control and support functions

These functions account for, check and oversee the lending business.

| Department | Roles | What they do in the system |
| --- | --- | --- |
| Finance / Accounting | Accountant, Provisioning Analyst | Post to the general ledger, reconcile, calculate ECL provisions |
| Compliance / AML | Compliance Officer, AML Analyst | Review sanctions and PEP hits, check related-party lending, confirm regulatory compliance |
| Regulatory Reporting | Reporting Officer | Produce statutory and supervisory reports |
| Internal Audit | Internal Auditor | Read-only access to every record and the audit trail |
| Senior Management | CEO, Board Risk Committee, Business Heads | Use dashboards and approve the largest exposures |

## 5. IT and system administration

These roles run and configure the system but should not touch business decisions.

| Role | What they do in the system |
| --- | --- |
| System Administrator | Manages users, roles and permissions, with no access to business data |
| Business Configuration Admin | Maintains products, workflows, the authority matrix and rule parameters, under maker-checker control |
| IT Operations / Support | Runs batch monitoring, incident handling and integrations |
| Information Security Officer | Reviews access, security logs and data masking policies |

## 6. External parties

These parties exchange data with the system or use a portal, but are not bank staff.

- **Borrowers and guarantors**, through the online portal or mobile app
- **Credit bureaus** (in Japan, for example, CIC, JICC and KSC)
- **Property appraisers and valuation firms**
- **Law firms and judicial scriveners** (司法書士) for mortgage registration
- **Collection agencies and servicers**
- **Insurance companies** for collateral and credit life insurance
- **Guarantee companies** (保証会社), very common in Japanese retail lending
- **Partner platforms** that refer customers or co-lend through APIs
- **Regulators**, who receive reports rather than logging in

## 7. System actors (non-human)

These automated components act in workflows and need their own service identities and audit trail.

- Rule and decision engine (auto-approve or decline)
- Rating and scoring engine
- Early-warning engine
- Batch scheduler (interest accrual, classification, revaluation reminders)
- Integration services (core banking, payments, general ledger)

## 8. Summary count

The middle office carries the most roles, because independent review and risk control need several separate layers.

| Group | Departments | Roles (approx.) |
| --- | --- | --- |
| Front office | 4–5 | 10–12 |
| Middle office (risk and credit) | 5–6 | 12–14 |
| Back office (operations) | 4–5 | 9–10 |
| Control and support | 4–5 | 6–8 |
| IT and administration | 1–2 | 4 |
| **Internal total** | **~15–20** | **~40** |

External parties add another 8–10 types.

## 9. Key design principles for roles

The system, not just policy, must enforce who can do what.

1. **Separate front and back.** The RM who originates a loan must never approve it, and approvers must never disburse it. The system enforces this through role conflicts.
2. **Use maker-checker** for every critical action: limit setup, disbursement, collateral release, write-off and configuration changes.
3. **Keep the authority matrix separate from roles.** A role defines which screens and actions a user can access. Authority defines how much they can approve, by amount, rating, product and collateral type. Store authority as configurable data, because it changes often.
4. **Add data scope on top of the role.** Two users with the same role may see different data: their own customers, their branch, their region, or the whole bank.
5. **Support delegation.** Approvers go on leave, so the system needs temporary authority delegation with an audit trail.
6. **Add compensating controls in small branches.** Where one person must hold two roles, require a second review or after-the-fact audit sampling.
