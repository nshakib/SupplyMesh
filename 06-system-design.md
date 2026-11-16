# 06. System Design

Common pipeline for all workflows: `NGINX → JWT guard → tenant guard (SET LOCAL app.org_id) → permission guard → use case (one DB transaction) → outbox event → response`. Async consumers handle notifications and scoring after commit.

---
## Workflow 1: Supplier registration and verification
```mermaid
sequenceDiagram
  actor PM as Procurement Manager
  participant API
  participant DB as PostgreSQL
  participant S3
  participant W as Worker
  PM->>API: POST /suppliers
  API->>DB: INSERT supplier (status pending, verification pending) + audit + outbox
  API-->>PM: 201 supplier
  PM->>API: POST /suppliers/:id/documents/upload-url
  API-->>PM: pre-signed PUT URL
  PM->>S3: upload file
  PM->>API: POST /suppliers/:id/documents (metadata, checksum)
  API->>DB: INSERT supplier_documents (review pending)
  API->>W: enqueue scan job
  W->>S3: fetch and scan
  W->>DB: UPDATE document scan result
  PM->>API: POST /suppliers/:id/verify
  API->>DB: BEGIN; check required docs valid; UPDATE verification verified, status active; INSERT audit, outbox; COMMIT
  W-->>PM: notification
```
- **Business logic:** unique tax ID per org; required document types come from org settings; documents must be scanned clean and unexpired.
- **Authorization:** `supplier:create`, `supplier:verify` (Procurement Manager/Owner); verifier ≠ creator is configurable.
- **Database ops:** inserts to suppliers and documents; update verification fields; audit and outbox in the same transaction.
- **Failures:** duplicate tax ID → 409. Upload succeeded but metadata call never came → orphan sweeper deletes unreferenced S3 keys. Scan fails → document `rejected`, verification blocked. Worker down → jobs wait in Redis and are processed on recovery.
- **Transactions:** verification is one transaction; file upload is outside the DB and made consistent by checksum plus sweeper.
- **Notifications:** supplier verified/rejected, document expiring in 30/7 days.

---
## Workflow 2: Purchase request and multi-level approval
```mermaid
sequenceDiagram
  actor E as Employee
  actor A1 as Dept Manager
  actor A2 as Procurement Manager
  participant API
  participant ENG as Approval Engine
  participant DB as PostgreSQL
  participant W as Worker
  E->>API: POST /purchase-requests/:id/submit
  API->>DB: BEGIN; SELECT request FOR UPDATE
  API->>ENG: select rule(cost, department)
  ENG->>DB: read active workflow and rules
  ENG-->>API: stages
  API->>DB: INSERT approval_steps snapshot; stage 1 pending, others waiting; UPDATE request pending_approval; audit; outbox; COMMIT
  W-->>A1: notify
  A1->>API: POST /approval-steps/:id/approve
  API->>DB: BEGIN; SELECT step FOR UPDATE; verify pending, approver, limit, not requester
  API->>DB: UPDATE step approved; activate next stage; COMMIT
  W-->>A2: notify
  A2->>API: POST /approval-steps/:id/approve
  API->>DB: last stage approved; request approved; audit; outbox; COMMIT
  W-->>E: request approved
```
- **Business logic:** rule chosen by cost band and department (department-specific rule beats generic). Example: under $500 → Dept Manager; $500–$5,000 → Procurement Manager; above $5,000 → Dept Manager, Procurement Manager, Finance Officer (all required, sequential). A stage with `mode=all` needs every listed approver; `any_one` completes on the first. Any rejection ends the request.
- **Authorization:** `request:create` for the requester and owner of the request; approver must be the assigned approver (or a delegate), have approval limit ≥ amount, and not be the requester.
- **Database ops:** `SELECT … FOR UPDATE` on request and step; snapshot insert; status updates; audit and outbox rows.
- **Failures:** two approvers click at once → row lock plus status check makes the second get 409. No matching rule → 422 with clear message, request stays draft. Approver leaves the org → steps reassigned by admin or escalation job. Policy edited mid-flight → no effect due to snapshot. Worker outage → notifications delayed, state remains correct.
- **Transactions:** submit and each decision are single transactions; optimistic `version` on the request detects stale clients.
- **Notifications:** next approver, requester on every decision, reminder at SLA (e.g. 24h), escalation at 48h.

---
## Workflow 3: Purchase order generation
```mermaid
sequenceDiagram
  actor PM as Procurement Manager
  participant API
  participant DB as PostgreSQL
  participant W as Worker
  PM->>API: POST /purchase-orders (requestId, supplierId, items)
  API->>DB: BEGIN
  API->>DB: SELECT request FOR UPDATE; check approved and no PO
  API->>DB: SELECT supplier; check verified and active
  API->>DB: UPDATE number_sequences (next PO number)
  API->>DB: INSERT purchase_orders (draft) and items; compute totals
  alt total beyond tolerance of estimate
    API->>DB: flag requires re-approval, create approval steps
  end
  API->>DB: UPDATE request ordered; audit; outbox; COMMIT
  API-->>PM: 201 draft PO
  PM->>API: POST /purchase-orders/:id/issue
  API->>DB: status issued, issued_at; outbox
  W-->>PM: PO document generated and emailed to supplier
```
- **Business logic:** one PO per approved request (MVP); supplier must be verified and active; totals computed server-side with `Money`; tolerance from org settings; PO numbers are per-org, sequential.
- **Authorization:** `po:create`, `po:issue`; tenant check on request and supplier.
- **Database ops:** locked read of request, insert PO and items, sequence increment, status update.
- **Failures:** request already ordered → unique constraint on `purchase_request_id` returns 409. Supplier suspended between draft and issue → issue re-checks and fails. Crash mid-transaction → full rollback, sequence untouched (sequence update is inside the transaction, so numbers stay gap-free at the cost of brief serialization per org).
- **Transactions:** create is atomic; issue is a separate guarded transition.
- **Notifications:** requester (ordered), supplier email with PO, Finance Officer (new PO to track).

---
## Workflow 4: Invoice verification and payment processing
```mermaid
sequenceDiagram
  actor F as Finance Officer
  participant API
  participant M as Match Service
  participant DB as PostgreSQL
  participant W as Worker
  F->>API: POST /invoices (poId, number, lines, totals)
  API->>DB: BEGIN; INSERT invoice recorded (unique per supplier+number)
  API->>M: match(invoice, PO, receipts)
  M->>DB: read PO items, received quantities, prior invoiced and paid
  M-->>API: result and discrepancies
  alt within tolerance
    API->>DB: status matched
  else discrepancy
    API->>DB: INSERT invoice_discrepancies; status disputed
  end
  API->>DB: audit; outbox; COMMIT
  F->>API: POST /invoices/:id/approve-payment
  F->>API: POST /payments (invoiceId, amount, idempotencyKey)
  API->>DB: BEGIN; SELECT invoice FOR UPDATE; check status, remaining balance
  API->>DB: INSERT payment; UPDATE invoice partially_paid or paid
  API->>DB: if fully paid and PO fully received then PO closed; audit; outbox; COMMIT
  W->>DB: recalculate supplier performance
  W-->>F: confirmation
```
- **Business logic:** three-way match compares invoice vs. PO price and quantity vs. goods received; tolerance (e.g. 2% or fixed amount) from settings; duplicate invoice numbers rejected; payments cannot exceed the outstanding balance; disputed invoices cannot be paid until discrepancies are resolved or waived (waiver needs a permission and reason).
- **Authorization:** `invoice:record`, `invoice:resolve`, `payment:record`; approver for payment may differ from the recorder (segregation of duties).
- **Database ops:** insert invoice and discrepancies; `FOR UPDATE` on invoice during payment; update statuses; PO closure check.
- **Failures:** double-click or client retry → idempotency key returns the original payment. Concurrent payments → invoice row lock serializes them. Match service sees stale receipts → match runs inside the same transaction snapshot (REPEATABLE READ for the check). Payment failed (when integrated with a bank later) → status `failed`, invoice balance unchanged, retry or reversal record rather than deleting.
- **Transactions:** record+match in one transaction; each payment is its own transaction; reversals are new rows, never edits.
- **Notifications:** discrepancy found (Procurement Manager and Finance), payment completed, overdue invoices (scheduled job).

---
## Bottlenecks and scaling

| Bottleneck | Cause | Mitigation |
|------------|-------|------------|
| Dashboard/analytics queries | Aggregations over growing spend data | Rollup tables and materialized views refreshed by jobs, Redis cache, read replica, later columnar store |
| Hot approval rows | Many approvers per large request | Row locks are short; per-request, not global |
| PO number sequence | Per-org serialization | Per-org counter row; contention only within one org; acceptable at procurement volumes |
| Notification spikes | Bulk submissions | Queue buffering, worker concurrency, provider rate limiting |
| Large audit/outbox tables | Append-only growth | Monthly partitions, BRIN indexes, archival, outbox purge after processing |
| DB connections | Many API instances | PgBouncer, pool limits |
| Noisy tenant | Shared schema | Per-tenant rate limits, statement timeouts, move large tenant to dedicated DB |
| API throughput | CPU on a single node | Stateless API containers scale horizontally behind NGINX/ALB; sessions are token-based |
| Document uploads | Large files through API | Direct-to-S3 pre-signed uploads |
