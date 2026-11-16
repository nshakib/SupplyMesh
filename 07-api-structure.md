# 07. API Structure Proposal

## Conventions
- Base path `/api/v1`, REST + JSON, OpenAPI 3 documented. The tenant is taken from the access token, **never from the URL or body**.
- Nouns for resources, verbs only for state transitions (`POST /resource/:id/approve`).
- **Pagination:** keyset (`?limit=25&cursor=...`) on large collections; filters via query (`?status=pending&department_id=...`); sorting via `sort=-created_at`.
- **Idempotency:** `Idempotency-Key` header required on payment and PO creation.
- **Concurrency:** `version` in body (or `If-Match`) on updates; stale write → 409.
- **Rate limiting:** per user and per organization (Redis), stricter on auth endpoints.
- **Validation:** DTO schemas (shared Zod contracts); unknown fields rejected.

### Response envelope
Success:
```json
{ "success": true, "message": "Purchase request submitted", "data": { "id": "..." } }
```
Error:
```json
{ "success": false, "message": "Validation failed", "errors": [ { "field": "estimatedCost", "message": "Must be >= 0" } ] }
```
List responses place `{ items, pageInfo: { nextCursor, limit } }` inside `data`.

### Status codes
200/201 success, 204 no content, 400 validation, 401 unauthenticated, 403 forbidden (permission or policy), 404 not found *(also returned for other tenants' resources, to avoid leaking existence)*, 409 conflict (state, duplicate, stale version), 422 business-rule violation, 429 rate limited.

## Endpoint catalogue

| Area | Method and path | Permission | Notes |
|------|-----------------|-----------|-------|
| **Auth** | POST /auth/register-organization | public | creates org, owner, seeded roles |
| | POST /auth/login, /auth/refresh, /auth/logout | public / cookie | rotating refresh |
| | POST /auth/switch-organization | authenticated | new token |
| | POST /auth/forgot-password, /reset-password, /verify-email | public | |
| **Organization** | GET/PATCH /organization | org:read / org:manage | settings, tolerances |
| | GET/POST/PATCH/DELETE /departments | dept:* | |
| | POST /invitations; POST /invitations/:token/accept | member:invite | |
| | GET/PATCH /members, /members/:id | member:manage | role, department, approval limit |
| | GET/POST/PATCH /roles; GET /permissions | role:manage | custom roles |
| **Suppliers** | GET/POST /suppliers; GET/PATCH /suppliers/:id | supplier:read/create/update | |
| | POST /suppliers/:id/verify, /reject, /suspend, /reactivate | supplier:verify / manage | |
| | POST /suppliers/:id/documents/upload-url; POST/GET /suppliers/:id/documents | supplier:update | pre-signed upload |
| | GET /suppliers/:id/performance | supplier:read | scores and trend |
| | GET/POST /supplier-categories | supplier:update | |
| **Requests** | GET/POST /purchase-requests; GET/PATCH /purchase-requests/:id | request:* (own or department scope) | |
| | POST /purchase-requests/:id/submit, /cancel | request:create | |
| | GET /purchase-requests/:id/approvals | request:read | step timeline |
| | GET/POST /purchase-requests/:id/comments | request:read | |
| **Approvals** | GET /approval-steps?mine=true&status=pending | approval:act | inbox |
| | POST /approval-steps/:id/approve, /reject | approval:act | body: comment |
| | POST /approval-steps/:id/delegate | approval:act | |
| | GET/POST/PATCH /approval-workflows; /approval-workflows/:id/rules | workflow:manage | rule and stage editor with overlap validation |
| | POST /approval-workflows/:id/simulate | workflow:manage | "what would happen for $X in dept Y" |
| **Purchase orders** | GET/POST /purchase-orders; GET/PATCH /purchase-orders/:id | po:* | |
| | POST /purchase-orders/:id/issue, /cancel, /close | po:issue | |
| | POST /purchase-orders/:id/receipts | po:receive | goods receipt |
| | GET /purchase-orders/:id/history | po:read | |
| **Invoices** | GET/POST /invoices; GET /invoices/:id | invoice:read/record | |
| | POST /invoices/:id/rematch, /approve-payment, /void | invoice:resolve | |
| | GET /invoices/:id/discrepancies; POST /discrepancies/:id/resolve | invoice:resolve | |
| **Payments** | GET/POST /payments; GET /invoices/:id/payments | payment:record | idempotent |
| | POST /payments/:id/reverse | payment:reverse | |
| **Analytics** | GET /analytics/spend?groupBy=department|supplier|month | analytics:read | |
| | GET /analytics/pending-requests, /approval-times | analytics:read | |
| | GET /analytics/supplier-performance | analytics:read | |
| **Platform** | GET /notifications; PATCH /notifications/:id/read | authenticated | |
| | GET /audit-logs?entityType=&entityId= | audit:read | owner and finance |
| | GET /health, /ready | public | for NGINX and orchestrator |

## Error and security notes
- Never return stack traces; each response carries a `requestId` also written to logs and audit entries.
- Authorization failures that would reveal another tenant's data return 404.
- Webhooks and integrations (future) will use signed payloads under `/api/v1/integrations/*`.
