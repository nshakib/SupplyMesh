# SupplyMesh: Global Procurement & Supplier Intelligence Platform

Design-first engineering blueprint. No application code; every decision is traced back to a business requirement.

| # | Document | Covers |
|---|----------|--------|
| 01 | [Product Feature Specification](01-product-feature-specification.md) | Features, roles, business rules, MoSCoW matrix, dependencies, limitations |
| 02 | [User Journey Diagrams](02-user-journeys.md) | Supplier onboarding, request approval, PO to invoice |
| 03 | [ER Diagram](03-er-diagram.md) | Complete entity relationship diagram |
| 04 | [Database Schema](04-database-schema.md) | Tables, keys, constraints, indexes, tenancy, growth |
| 05 | [Architecture](05-architecture.md) | High-level diagram, frontend, backend, auth, storage, jobs, cache, deployment |
| 06 | [System Design](06-system-design.md) | Four critical workflows, bottlenecks, scaling |
| 07 | [API Structure](07-api-structure.md) | REST conventions and endpoint catalogue |
| 08 | [ADRs](08-adrs.md) | 10 architecture decision records |
| 09 | [MVP vs Future Scope](09-mvp-vs-future-scope.md) | Phasing and rationale |
| 10 | [Risk & Scalability](10-risk-and-scalability.md) | Risk register, scaling path, limitations |
| 11 | [Presentation Outline](11-presentation-outline.md) | Video talking points (reasoning, not reading) |

Diagrams use Mermaid and render natively on GitHub.

## Business understanding (summary)
Medium-to-large organizations buy from many suppliers across departments and countries. Their pain is **loss of control**: purchases happen without policy, approvals are slow or informal, invoices do not match what was ordered, and nobody can say which suppliers are reliable. The core problem is therefore *governed spend*: every purchase should be requested, approved by the right person for its value, ordered from a verified supplier, reconciled against its invoice, and recorded immutably.

Three consequences shape the whole design:
1. **Money and approvals demand integrity**, so a relational database with transactions.
2. **Policies differ per organization**, so approval rules are data, not code.
3. **Many organizations share one product**, so tenant isolation is a first-class, layered concern.
