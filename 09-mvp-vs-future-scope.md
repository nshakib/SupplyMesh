# 09. MVP vs Future Scope

## Guiding principle
The MVP must prove one complete, governed loop: **supplier → request → approval → PO → receipt → invoice → payment → audit**, for multiple isolated organizations. Anything that does not strengthen that loop, or that depends on data the loop has not produced yet, is postponed.

## Phase 1: MVP
| Included | Why it is in |
|----------|--------------|
| Organizations, users, invitations, departments | Tenancy and identity are the foundation of every other feature |
| RBAC with five seeded roles | Permission boundaries are an explicit requirement; seeded roles avoid building an editor first |
| Supplier registration, documents, verification, status | POs may only use verified suppliers, so this gates the loop |
| Purchase requests | The entry point of the procurement lifecycle |
| Configurable approval rules (amount bands, sequential stages) | The core differentiator and the hardest part to retrofit |
| Purchase orders, items, goods receipts | Needed for order tracking and the three-way match |
| Invoices, three-way match, discrepancies, payments | Prevents overpayment, which is the main financial value |
| Audit log | Cheap to build early, impossible to backfill |
| Basic analytics (spend by department/supplier, pending, approval time) | Directly requested and computable from loop data |
| Email notifications for approvals | Approvals stall without them |

## Phase 2: Should-have (after first customers)
Custom role editor, approval delegation and SLA escalation, supplier performance scoring and trends, request comments/attachments, in-app notification centre, CSV/PDF export, org-level MFA enforcement.

## Phase 3: Future expansion
Risk indicators and scoring (document expiry, late-delivery patterns, external sanctions/financial feeds), RFQ and bidding, contract lifecycle management, budgets and spend commitments, multi-currency with FX, ERP/accounting integrations (webhooks, CSV import), SSO/SAML, supplier self-service portal, mobile approvals, ML anomaly and duplicate detection, dedicated-tenant deployments, data warehouse for analytics.

## Why postponed items wait
| Postponed | Reason |
|-----------|--------|
| Supplier performance scoring | Needs a history of receipts and invoices; MVP collects the raw performance records so scoring can be computed retroactively. |
| External risk feeds | Depend on third-party contracts and data quality; internal indicators come first. |
| RFQ/contracts | Separate lifecycles with their own states; adding them early dilutes the core loop. |
| Budgets | Require commitment accounting across requests and POs; valuable but a major rule-engine extension. |
| Multi-currency | FX adds rounding and reporting complexity; currency codes are stored from day one so it can be added without migration pain. |
| SSO / ERP | Enterprise-sales features; the auth and integration layers are built behind interfaces. |
| Microservices, Kubernetes, search engine | No evidence of need; would add operational cost without product value. |

## Extensibility hooks built into the MVP
Currency columns, snapshot fields, outbox events, module facades, tenant-routing abstraction, approver-type strategies, and `settings` jsonb on organizations.
