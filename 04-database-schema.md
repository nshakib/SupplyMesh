# 04. Database Schema Documentation

Engine: **PostgreSQL 16**, accessed via **Drizzle ORM**. Conventions: `uuid` PKs (UUIDv7 for index locality), `timestamptz` everywhere, `numeric(14,2)` for money, `text` + `CHECK` (or Postgres enums) for statuses, soft delete only where history matters (`suppliers`, `memberships`), `created_at/updated_at` on all tables, `version int` on tables needing optimistic locking.

## 1. Table definitions

### Identity and access
| Table | Columns (PK **bold**, FK → ) | Key constraints |
|-------|------------------------------|-----------------|
| organizations | **id**, name, slug, base_currency char(3), settings jsonb (tolerances, defaults), status, created_at | UNIQUE(slug) |
| users | **id**, email citext, password_hash, full_name, email_verified_at, mfa_secret, last_login_at | UNIQUE(email) |
| departments | **id**, organization_id → organizations, name, manager_membership_id → memberships, monthly_budget numeric NULL | UNIQUE(organization_id, name) |
| roles | **id**, organization_id → organizations, name, is_system bool | UNIQUE(organization_id, name) |
| permissions | **id**, code text, description | UNIQUE(code) (global catalogue) |
| role_permissions | **role_id** → roles, **permission_id** → permissions | composite PK |
| memberships | **id**, organization_id, user_id → users, department_id → departments, role_id → roles, approval_limit numeric NULL, status (invited/active/suspended) | UNIQUE(organization_id, user_id); composite FK (organization_id, department_id) |
| invitations | **id**, organization_id, email, role_id, token_hash, expires_at, accepted_at | UNIQUE(token_hash) |

### Suppliers
| Table | Columns | Constraints |
|-------|---------|-------------|
| supplier_categories | **id**, organization_id, name | UNIQUE(organization_id, name) |
| suppliers | **id**, organization_id, category_id → supplier_categories, name, legal_name, tax_id, country char(2), contact jsonb, payment_terms_days, status (active/suspended/blacklisted/inactive), verification_status (pending/verified/rejected), verified_by → memberships, verified_at, deleted_at | UNIQUE(organization_id, tax_id) WHERE deleted_at IS NULL |
| supplier_documents | **id**, organization_id, supplier_id → suppliers, type, storage_key, checksum, mime_type, size_bytes, expires_at, review_status, reviewed_by | CHECK(size_bytes > 0); index on expires_at |

### Approvals
| Table | Columns | Constraints |
|-------|---------|-------------|
| approval_workflows | **id**, organization_id, name, is_default, is_active, version | one default per org (partial unique index) |
| approval_rules | **id**, organization_id, workflow_id → approval_workflows, department_id NULL, min_amount, max_amount NULL, priority | CHECK(max_amount IS NULL OR max_amount > min_amount); EXCLUDE constraint on numrange to forbid overlapping bands per workflow+department |
| approval_rule_stages | **id**, organization_id, rule_id → approval_rules, stage_order, approver_type (role/user/dept_manager), approver_role_id NULL, approver_membership_id NULL, mode (any_one/all) | UNIQUE(rule_id, stage_order); CHECK exactly the matching approver column is set |
| approval_steps | **id**, organization_id, purchase_request_id → purchase_requests, rule_stage_id (snapshot ref), stage_order, approver_membership_id, status (waiting/pending/approved/rejected/skipped), comment, decided_at, due_at | UNIQUE(purchase_request_id, stage_order, approver_membership_id); CHECK(approver ≠ requester) enforced in service plus trigger |

### Procurement
| Table | Columns | Constraints |
|-------|---------|-------------|
| purchase_requests | **id**, organization_id, request_number, requester_id → memberships, department_id, item_name, description, quantity numeric, estimated_cost numeric, currency, priority (low/medium/high/urgent), required_by date, status (draft/pending_approval/approved/rejected/cancelled/ordered), workflow_id, rule_id (snapshot), current_stage, version, submitted_at | CHECK(quantity > 0), CHECK(estimated_cost >= 0); UNIQUE(organization_id, request_number) |
| purchase_orders | **id**, organization_id, po_number, purchase_request_id UNIQUE NULL, supplier_id → suppliers, created_by, currency, subtotal, tax_total, total_amount, status (draft/issued/partially_received/received/closed/cancelled), expected_delivery, issued_at, version | UNIQUE(organization_id, po_number); CHECK(total_amount = subtotal + tax_total) |
| purchase_order_items | **id**, organization_id, purchase_order_id, line_no, description, quantity, unit_price, line_total, received_qty | UNIQUE(purchase_order_id, line_no); CHECK(quantity > 0, received_qty <= quantity); line_total generated column |
| goods_receipts | **id**, organization_id, purchase_order_id, received_by, received_at, notes | |
| goods_receipt_items | **id**, goods_receipt_id, po_item_id, quantity_received, condition | CHECK(quantity_received > 0) |

### Finance
| Table | Columns | Constraints |
|-------|---------|-------------|
| invoices | **id**, organization_id, purchase_order_id, supplier_id, invoice_number, issue_date, due_date, subtotal, tax_total, total_amount, status (recorded/matched/disputed/approved/partially_paid/paid/void), matched_at, recorded_by, version | UNIQUE(organization_id, supplier_id, invoice_number) blocks duplicate invoices |
| invoice_discrepancies | **id**, organization_id, invoice_id, type (price/quantity/total/duplicate/unordered_item), expected numeric, actual numeric, status (open/resolved/waived), resolved_by, resolution_note | |
| payments | **id**, organization_id, invoice_id, amount, method, reference, paid_at, status (scheduled/completed/failed/reversed), recorded_by, idempotency_key | CHECK(amount > 0); UNIQUE(organization_id, idempotency_key) |

### Intelligence and platform
| Table | Columns | Constraints |
|-------|---------|-------------|
| supplier_performance_records | **id**, organization_id, supplier_id, purchase_order_id, metric (on_time_delivery/fill_rate/discrepancy), value numeric, recorded_at | append-only |
| supplier_scores | **id**, organization_id, supplier_id, period (month), delivery_score, completion_rate, discrepancy_rate, reliability_score, risk_level, spend_total | UNIQUE(supplier_id, period) |
| audit_logs | **id** bigserial, organization_id, actor_membership_id, entity_type, entity_id, action, before jsonb, after jsonb, ip, request_id, created_at | append-only; UPDATE/DELETE revoked; partitioned by month |
| notifications | **id**, organization_id, recipient_membership_id, type, payload jsonb, read_at | |
| outbox_events | **id**, organization_id, aggregate_type, aggregate_id, event_type, payload jsonb, processed_at | partial index WHERE processed_at IS NULL |
| number_sequences | **organization_id, kind**, next_value | gap-free per-org numbering for PO/request numbers |

## 2. Important constraints
- **Integrity:** FKs with `ON DELETE RESTRICT` on financial chains; no cascade deletes of money records.
- **Composite tenant FKs:** child tables reference `(organization_id, id)` of the parent so a row can never point at another tenant's parent.
- **Money:** `numeric`, never float; sums verified by CHECK and by service-layer invariants.
- **State machines:** CHECK on status values; legal transitions enforced in domain services and covered by tests.
- **Duplicate prevention:** unique invoice per supplier, unique tax ID per org, idempotency key on payments.
- **Overlap prevention:** `EXCLUDE USING gist` on approval bands.

## 3. Indexing strategy
| Index | Serves |
|-------|--------|
| (organization_id, status, created_at DESC) on purchase_requests, purchase_orders, invoices | list and dashboard queries |
| (approver_membership_id, status) WHERE status = 'pending' on approval_steps | "my pending approvals" inbox |
| (organization_id, supplier_id, created_at) on purchase_orders, invoices | supplier spend and history |
| (organization_id, department_id, submitted_at) on purchase_requests | department spend |
| (expires_at) WHERE review_status = 'approved' on supplier_documents | expiry job |
| (organization_id, entity_type, entity_id, created_at) on audit_logs | entity timeline |
| BRIN on audit_logs.created_at | cheap range scans on huge append-only table |
| partial index on outbox_events WHERE processed_at IS NULL | relay worker polling |
| GIN trigram on suppliers.name | fuzzy supplier search |

Rule: **organization_id is the leading column** in almost every composite index, which matches the tenant filter on every query.

## 4. Normalization decisions
- Schema is in **3NF** for transactional data: suppliers, items, receipts and payments are separate entities with FKs.
- **Deliberate denormalization:** `purchase_orders.total_amount`, `purchase_order_items.line_total`, `supplier_scores` (rollups), and the snapshot columns on `purchase_requests` (workflow/rule) and `approval_steps`. Reason: totals are read constantly and snapshots must not change when configuration changes. Consistency is protected by CHECK constraints and transactional updates.
- **jsonb** only for genuinely schemaless data (org settings, supplier contact blocks, audit before/after, notification payloads). Anything queried or joined on is a column.

## 5. Multi-tenant isolation strategy (defence in depth)
1. **Shared database, shared schema, `organization_id` on every tenant table.**
2. **Application layer:** a request-scoped `TenantContext` extracted from the JWT; repositories accept only a tenant-bound DB handle, so a query without a tenant filter cannot be written by feature code.
3. **Database layer:** Postgres **Row-Level Security** with `USING (organization_id = current_setting('app.org_id')::uuid)`. Each transaction runs `SET LOCAL app.org_id = ...`. The app connects as a non-owner role that cannot bypass RLS.
4. **Composite FKs** prevent cross-tenant references.
5. **Storage:** S3 keys prefixed `org/{orgId}/...`; access only through short-lived signed URLs after an authorization check.
6. **Tests:** automated cross-tenant leakage tests on every endpoint.

## 6. Handling growth
- **Now:** correct indexes, keyset pagination, read-optimized rollup tables.
- **Next:** pooled connections (PgBouncer), read replica for analytics, monthly **partitioning** of `audit_logs`, `outbox_events`, `notifications`, and later `invoices`/`payments` by hash of organization_id or by time.
- **Large tenants:** move the tenant to a dedicated database (the tenant-context abstraction makes routing per org possible) without schema changes.
- **Archival:** closed POs/invoices older than N years move to cold partitions or S3 Parquet; audit logs archived, never deleted.
- **Analytics:** materialized views refreshed by background jobs; if needed, ship to a columnar store later.

## 7. Why this structure, and alternatives considered
| Choice | Alternatives considered | Why this one |
|--------|------------------------|--------------|
| Normalized relational model | Document model (MongoDB), event-sourced ledger | Procurement is dominated by relations and invariants (PO ↔ invoice ↔ payment). Document stores would push integrity into application code. Event sourcing adds complexity before audit needs justify it; the audit log plus snapshots gives most of the value. |
| Shared schema + RLS | Schema per tenant, DB per tenant | Per-tenant schemas multiply migrations and connections; hundreds of tenants stay cheap with one schema. Dedicated DB remains an escape hatch. |
| Rules + stages + step snapshots | Hard-coded thresholds; BPMN/JSON engine | Hard-coding fails the "configurable per org" requirement; a general BPM engine is overkill. Relational rules are queryable, validated by constraints, and enough for amount-band, department-scoped chains. |
| Separate goods receipts | Just `received_qty` on items | Receipts give timing and partial deliveries, which delivery-performance scoring needs. |
