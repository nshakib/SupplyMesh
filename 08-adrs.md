# 08. Architecture Decision Records

Each ADR: **Problem, Alternatives, Decision, Reason, Advantages, Disadvantages, Trade-offs.**

---
## ADR-001: PostgreSQL as the primary database
- **Problem:** Store organizations, suppliers, requests, orders, invoices and payments with strong integrity.
- **Alternatives:** MongoDB; MySQL; a hybrid (Postgres + document store).
- **Decision:** PostgreSQL.
- **Reason:** SupplyMesh data is heavily relational (organization → supplier → PO → invoice → payment) and money movements need multi-row ACID transactions, foreign keys, CHECK/EXCLUDE constraints and row locks. Postgres adds Row-Level Security (needed for tenancy), partial indexes, jsonb for the few flexible fields, partitioning and materialized views for analytics. MySQL could do much of this but lacks RLS and exclusion constraints; MongoDB would move invariants into application code.
- **Advantages:** integrity at the database level, one engine for OLTP and moderate analytics, mature tooling.
- **Disadvantages:** vertical scaling first; horizontal scaling needs partitioning or sharding work later.
- **Trade-offs:** we give up schema-less flexibility, which this domain barely needs, for guarantees it needs a lot.

## ADR-002: Drizzle ORM instead of Prisma
- **Problem:** Choose a data-access layer that works with a transaction-heavy, RLS-based, SQL-intensive design.
- **Alternatives:** Prisma, TypeORM, raw SQL with a query builder (Kysely).
- **Decision:** Drizzle ORM with drizzle-kit migrations.
- **Reason:** We need `SET LOCAL app.org_id` inside each transaction, explicit transactions with `FOR UPDATE`, composite keys, partial/GIN/BRIN indexes, exclusion constraints and hand-tuned analytics queries. Drizzle's schema is TypeScript, SQL-shaped and close to what runs on the wire; raw `sql` fragments are first-class. It is also lightweight (no query engine binary), which simplifies Docker images.
- **Advantages:** type-safe SQL, predictable queries, easy RLS and locking, small footprint.
- **Disadvantages:** younger ecosystem, fewer built-in conveniences (no automatic relation includes like Prisma's), migrations need more manual review for advanced Postgres features.
- **Trade-offs:** Prisma offers a gentler developer experience and a polished client; we trade that for control over SQL, which this domain rewards. If the team were less SQL-comfortable, Prisma would be a defensible choice.

## ADR-003: Modular monolith instead of microservices
- **Problem:** Pick a deployable structure for a new SaaS with a small team and a transaction-centered domain.
- **Alternatives:** Microservices; SOA; plain layered monolith.
- **Decision:** Modular monolith in NestJS with enforced module boundaries.
- **Reason:** Approve → create PO → invoice → pay spans modules and must be atomic. In-process calls inside one DB transaction avoid sagas and compensation logic. No module has independent scaling or team ownership yet.
- **Advantages:** simple deploys, easy debugging, low infrastructure cost, strong consistency.
- **Disadvantages:** a hot module cannot scale alone; discipline is needed to avoid boundary erosion; one bad deploy affects all.
- **Trade-offs:** we accept coarse scaling now. Boundaries via facades and domain events (checked with lint rules such as dependency-cruiser) keep extraction possible when evidence appears.

## ADR-004: Redis is required, but scoped
- **Problem:** Decide whether Redis belongs in the MVP.
- **Alternatives:** No Redis (DB-backed queue such as pg-boss, in-memory cache); Redis; managed queue (SQS).
- **Decision:** Use Redis for BullMQ queues, permission/config cache and rate limiting. Not used as a source of truth.
- **Reason:** Notifications, document scans, scoring, SLA escalations and exports must not block requests; retries and delayed/repeatable jobs are core. Rate limiting and permission lookups run on every request.
- **Advantages:** reliable retries/scheduling, sub-millisecond lookups, fewer DB reads.
- **Disadvantages:** another stateful component to run and secure; cache invalidation risk.
- **Trade-offs:** pg-boss would remove Redis for the MVP, and is a valid cost-saving alternative; Redis is chosen because the specified ecosystem includes it, and cache plus rate limiting benefit anyway. Rule: if Redis is lost, the system degrades (slower, delayed jobs) but loses no business data, because the outbox lives in Postgres.

## ADR-005: Multi-tenancy via shared schema, `organization_id` and Row-Level Security
- **Problem:** Isolate organizations strictly while keeping operations simple.
- **Alternatives:** Database per tenant; schema per tenant; shared schema with application-only filtering.
- **Decision:** Shared database and schema, `organization_id` on every table, composite tenant FKs, tenant-bound repositories, **plus** Postgres RLS.
- **Reason:** Requirement is strict isolation across potentially many organizations. App-level filtering alone is one forgotten `WHERE` away from a breach, so RLS adds a database-enforced backstop. Per-tenant schemas/databases multiply migrations, connections and cost.
- **Advantages:** single migration path, efficient resource use, cross-tenant leakage prevented at two layers.
- **Disadvantages:** noisy neighbours, RLS adds small planning overhead, backup/restore of one tenant is harder.
- **Trade-offs:** enterprise customers wanting physical isolation get a **dedicated database** option later via tenant-based connection routing.

## ADR-006: Approval workflows stored as configurable rules with snapshot steps
- **Problem:** Each organization has its own approval policy that must be editable without deployments, yet in-flight approvals must stay stable and auditable.
- **Alternatives:** Hard-coded thresholds; JSON blob per organization; BPMN/workflow engine (Camunda, Temporal); relational rules (chosen).
- **Decision:** Tables `approval_workflows → approval_rules (amount band, optional department) → approval_rule_stages`; at submission the engine materializes `approval_steps` for that request.
- **Reason:** Amount-band, department-scoped, sequential/any/all chains cover stated needs. Relational storage allows constraints (no overlapping bands), validation and a simulate endpoint. Snapshotting keeps history truthful when policy later changes.
- **Advantages:** configurable, queryable inbox, auditable, no external engine.
- **Disadvantages:** cannot express arbitrary branching/parallel graphs; evolving rule types requires schema/strategy updates.
- **Trade-offs:** A full workflow engine would be more general but adds operational weight and learning cost we cannot justify. The approver-type strategy interface lets us extend conditions (category, supplier risk) later.

## ADR-007: PO and invoice consistency through three-way matching and transactional invariants
- **Problem:** Ensure invoices and payments never exceed what was ordered and received.
- **Alternatives:** Two-way match (PO vs invoice) only; manual review; eventual-consistency reconciliation job.
- **Decision:** Synchronous three-way match (request/PO/receipt vs invoice) at invoice recording, with tolerance settings; payment and invoice status updates run under row locks in one transaction; discrepancies are stored as first-class rows.
- **Reason:** Overpayment is the main financial risk in procurement; it must be prevented at write time, not discovered later. CHECK constraints, unique invoice numbers and idempotency keys back up the logic.
- **Advantages:** strong correctness, explainable disputes, audit trail.
- **Disadvantages:** row locking adds contention on hot invoices; rules (tolerances) need careful configuration.
- **Trade-offs:** a stricter synchronous check slightly slows the write path, accepted because invoice volume is low relative to the cost of an error.

## ADR-008: Background jobs with BullMQ and the transactional outbox
- **Problem:** Send notifications and run derived computations without losing events or blocking requests.
- **Alternatives:** Publish directly after commit; cron-only polling; Kafka/RabbitMQ; SQS.
- **Decision:** Write events to an `outbox_events` table inside the business transaction; a relay moves them to BullMQ queues; handlers are idempotent with retries and a dead-letter queue.
- **Reason:** Publishing after commit can lose events on a crash; publishing before commit can announce things that roll back. The outbox gives at-least-once delivery tied to the commit.
- **Advantages:** no lost notifications, simple ops, replayable.
- **Disadvantages:** extra table and relay process; consumers must be idempotent; slight delivery delay.
- **Trade-offs:** Kafka-class brokers are far more than the current volume needs.

## ADR-009: JWT access tokens with rotating refresh tokens, and RBAC plus policy checks
- **Problem:** Authenticate users who may belong to several organizations and authorize at action and record level.
- **Alternatives:** Server-side sessions only; long-lived JWTs; external IdP (Auth0/Cognito) from day one.
- **Decision:** Short-lived JWT (active org + membership), rotating hashed refresh tokens in httpOnly cookies; RBAC via `role_permissions`, with policy checks (ownership, department scope, approval limit) in use cases.
- **Reason:** Stateless API scaling, quick revocation via refresh rotation, per-organization context in the token. Pure RBAC cannot express "only approve up to your limit" or "not your own request", so a policy layer is required.
- **Advantages:** horizontal scalability, clear separation of permission vs. policy.
- **Disadvantages:** access tokens stay valid until expiry (mitigated by short TTL); more code than an IdP.
- **Trade-offs:** an external IdP reduces build effort and eases SSO later; deferred to keep control of multi-membership logic. The auth module sits behind an interface to allow swapping.

## ADR-010: Deployment on Docker Compose and NGINX first, AWS-managed services as growth path
- **Problem:** Deploy economically now without blocking future scale.
- **Alternatives:** Kubernetes from the start; serverless; PaaS.
- **Decision:** Multi-container Docker Compose (nginx, web, api, worker, postgres, redis) on a VPS/EC2 with GitHub Actions CI/CD; migrate to ECS Fargate + RDS + ElastiCache when load or compliance requires it.
- **Reason:** A small team gains little from Kubernetes' operational overhead. Containers keep the application portable, so the move to AWS-managed services is a configuration change, not a rewrite.
- **Advantages:** cheap, understandable, reproducible environments.
- **Disadvantages:** single-host failure risk, manual scaling and patching in the early stage.
- **Trade-offs:** mitigated with managed Postgres or tested backups/PITR from day one, health checks and automated restarts.
