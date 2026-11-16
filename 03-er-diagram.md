# 03. Entity Relationship Diagram

Every tenant-owned table carries `organization_id`. Rows are shown with key columns only; full definitions are in document 04.

```mermaid
erDiagram
  ORGANIZATIONS ||--o{ DEPARTMENTS : has
  ORGANIZATIONS ||--o{ MEMBERSHIPS : has
  USERS ||--o{ MEMBERSHIPS : joins
  DEPARTMENTS ||--o{ MEMBERSHIPS : groups
  ORGANIZATIONS ||--o{ ROLES : defines
  ROLES ||--o{ ROLE_PERMISSIONS : grants
  PERMISSIONS ||--o{ ROLE_PERMISSIONS : included_in
  ROLES ||--o{ MEMBERSHIPS : assigned_to

  ORGANIZATIONS ||--o{ SUPPLIERS : manages
  SUPPLIER_CATEGORIES ||--o{ SUPPLIERS : classifies
  SUPPLIERS ||--o{ SUPPLIER_DOCUMENTS : has

  ORGANIZATIONS ||--o{ APPROVAL_WORKFLOWS : configures
  APPROVAL_WORKFLOWS ||--o{ APPROVAL_RULES : contains
  APPROVAL_RULES ||--o{ APPROVAL_RULE_STAGES : defines

  MEMBERSHIPS ||--o{ PURCHASE_REQUESTS : requests
  DEPARTMENTS ||--o{ PURCHASE_REQUESTS : owns
  APPROVAL_WORKFLOWS ||--o{ PURCHASE_REQUESTS : governs
  PURCHASE_REQUESTS ||--o{ APPROVAL_STEPS : requires
  MEMBERSHIPS ||--o{ APPROVAL_STEPS : decides

  PURCHASE_REQUESTS ||--o| PURCHASE_ORDERS : fulfilled_by
  SUPPLIERS ||--o{ PURCHASE_ORDERS : receives
  PURCHASE_ORDERS ||--|{ PURCHASE_ORDER_ITEMS : contains
  PURCHASE_ORDERS ||--o{ GOODS_RECEIPTS : receives
  GOODS_RECEIPTS ||--|{ GOODS_RECEIPT_ITEMS : lists
  PURCHASE_ORDER_ITEMS ||--o{ GOODS_RECEIPT_ITEMS : received_as

  PURCHASE_ORDERS ||--o{ INVOICES : billed_by
  SUPPLIERS ||--o{ INVOICES : issues
  INVOICES ||--o{ INVOICE_DISCREPANCIES : flags
  INVOICES ||--o{ PAYMENTS : settled_by

  SUPPLIERS ||--o{ SUPPLIER_PERFORMANCE_RECORDS : scored_in
  PURCHASE_ORDERS ||--o{ SUPPLIER_PERFORMANCE_RECORDS : evidences

  ORGANIZATIONS ||--o{ AUDIT_LOGS : records
  ORGANIZATIONS ||--o{ NOTIFICATIONS : sends
  ORGANIZATIONS ||--o{ OUTBOX_EVENTS : emits

  ORGANIZATIONS {
    uuid id PK
    text name
    text slug UK
    char3 base_currency
    jsonb settings
  }
  USERS {
    uuid id PK
    citext email UK
    text password_hash
    text full_name
  }
  MEMBERSHIPS {
    uuid id PK
    uuid organization_id FK
    uuid user_id FK
    uuid department_id FK
    uuid role_id FK
    numeric approval_limit
    text status
  }
  SUPPLIERS {
    uuid id PK
    uuid organization_id FK
    uuid category_id FK
    text name
    text tax_id
    text status
    text verification_status
  }
  PURCHASE_REQUESTS {
    uuid id PK
    uuid organization_id FK
    uuid requester_id FK
    uuid department_id FK
    numeric estimated_cost
    text priority
    text status
  }
  APPROVAL_STEPS {
    uuid id PK
    uuid purchase_request_id FK
    int stage_order
    uuid approver_id FK
    text status
  }
  PURCHASE_ORDERS {
    uuid id PK
    uuid organization_id FK
    uuid supplier_id FK
    text po_number
    numeric total_amount
    text status
  }
  INVOICES {
    uuid id PK
    uuid purchase_order_id FK
    text invoice_number
    numeric total_amount
    text status
  }
  PAYMENTS {
    uuid id PK
    uuid invoice_id FK
    numeric amount
    text status
  }
  AUDIT_LOGS {
    bigint id PK
    uuid organization_id FK
    text entity_type
    uuid entity_id
    text action
    jsonb diff
  }
```

## Relationship explanations
- **Users are global; memberships are tenant-scoped.** One person can belong to several organizations with different roles/departments. Every authorization decision resolves through a membership, never through the user alone.
- **Roles belong to an organization** (custom roles); `permissions` is a global catalogue of capabilities (`request:create`, `po:issue`, …) mapped by `role_permissions`.
- **Workflow definition vs. instance:** `approval_workflows → approval_rules → approval_rule_stages` is the configurable template. `approval_steps` is the frozen per-request instance.
- **Request to PO is 1 to 0..1** in MVP (one approved request yields at most one PO). Relaxing to many is a unique-constraint change.
- **PO to invoices is 1 to many** (partial billing). **Invoice to payments is 1 to many** (instalments).
- **Goods receipts** are separate from PO items so partial deliveries and received quantities (needed for the 3-way match and delivery KPIs) are first-class facts.
- **Performance records** are derived, append-only facts per PO event (on-time, complete, discrepancy), rolled up into period scores.
