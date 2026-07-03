# TODO

Feature checklist for Codex, organized by milestone. Within each milestone, tasks are grouped by component. Check off as work lands.

---

## Design phase — core ADR sequence

- [x] ADR-0001: Schema-as-data over codegen
- [x] ADR-0002: Search backend per entity type
- [x] ADR-0003: Single generic table, JSON upsert
- [x] ADR-0004: Entity envelope model
- [x] ADR-0005: Draft-first workflow persistence
- [x] ADR-0006: Event contract versioning
- [x] ADR-0007: Subscriber Registry ownership
- [x] ADR-0008: Message broker — RabbitMQ
- [x] ADR-0009: Transactional outbox for promotion event emission

Core architectural direction, including both implementation-risk gaps ADR-0007 left open (broker choice, promotion dual-write), is settled as of ADR-0009. See `ARCHITECTURE.md` §10 for the phase-2 concerns deliberately deferred out of this sequence (registry domain separation, event registry model, authorization, cross-entity references, rollback, observability, multi-region) — pick these up opportunistically as their triggers hit, not as a blocking prerequisite to starting Milestone 0.

---

## Milestone 0 — Workspace bootstrap

- [ ] Initialize Cargo workspace (`codex/`) with empty crates: `common`, `registry`, `gateway`, `data-service`, `workflow`, `search`, `distribution`
- [ ] Shared `Result`/error type in `common` (thiserror-based, per-crate error enums convertible into a top-level error)
- [ ] Shared entity envelope type in `common` (id, entity_type, version, status, payload, created_at, updated_at)
- [ ] Structured logging/tracing setup shared across crates
- [ ] CI pipeline: build, clippy, test on push (reuse pattern from existing release workflow experience)
- [ ] Local dev environment (docker-compose: Postgres, Redis, RabbitMQ, Elasticsearch)
- [ ] `outbox_events` table + outbox relay skeleton in `common`/`data-service` (ADR-0009), adapting AnvilNotify's outbox worker (adaptive poll backoff, stale lock reaper)

---

## Milestone 1 — Entity Type Registry (material as first entity type)

- [ ] Define registry schema config format (JSON Schema vs custom DSL) — decision doc
- [ ] Registry storage (Postgres table: entity_type, schema_version, schema_def, workflow_def, index_def, ai_access_flags)
- [ ] Schema definition CRUD (register, update with versioning, deprecate an entity type)
- [ ] Field-level validation rules (required, type, enum, regex, nested objects)
- [ ] Workflow definition format (states, transitions, allowed roles per transition)
- [ ] Index/search definition format (which fields are searchable, mapping type, embedding on/off)
- [ ] Subscriber/distribution profile format (delivery method, priority tier, SLA) — schema only, wiring comes in Milestone 4
- [ ] AI-access flags per entity type (read-only / read-write / excluded from MCP)
- [ ] Registry versioning: schema migrations for an existing entity type without breaking stored data
- [ ] Register "material" as the first entity type — 1:1 mapping of existing mat schema
- [ ] Unit tests: schema validation edge cases, versioning conflicts
- [ ] Registry admin API (internal — for maintainers to add/edit entity types without redeploying)

---

## Milestone 2 — Generic Data Service

- [ ] CRUD endpoints (create, read, update, soft-delete) operating on `serde_json::Value` payload + envelope
- [ ] Boundary validation against registry schema for the given entity_type
- [ ] Postgres storage: single generic table (id, entity_type, payload JSONB, version, status, timestamps)
- [ ] GIN index on payload for query performance
- [ ] Table partitioning by entity_type (+ secondary key per registry config)
- [ ] Optimistic concurrency / version conflict handling on update
- [ ] Batch read/write API
- [ ] Migrate material's existing CRUD traffic onto the generic service
- [ ] Load test at material's current volume (26M rows) before cutover
- [ ] Integration tests per entity type schema (using material + a synthetic test entity type)

---

## Milestone 3 — Generic Workflow Service (draft-first, per ADR-0005)

- [ ] `entity_drafts` table (id, entity_type, business_key, target_entity_id, base_version, schema_version, workflow_state, payload, submitted_by, timestamps)
- [ ] State machine engine (generic — states/transitions loaded from registry per entity_type)
- [ ] Approval action API operating on drafts (submit, approve, reject, request changes)
- [ ] Role/permission check per transition (who can approve what, from registry config)
- [ ] Promotion transaction: base_version staleness check → final schema validation → upsert into `entities` (set `source_draft_id`, increment `version`) → remove/archive draft → write `EntityCommitted` outbox row, all in one Postgres transaction (ADR-0009); outbox relay publishes to RabbitMQ (ADR-0008) after commit
- [ ] Single-open-draft-per-target-entity enforcement (concurrent draft conflict rule, ADR-0005)
- [ ] Change request types generalized (replaces material's A/C/E/U/L/R enum) — per entity type, configurable
- [ ] Post-promotion transitions (e.g. approved → archived) operate directly on `entities.workflow_state`, no further draft cycle
- [ ] `DraftSubmitted` / `DraftRejected` / `EntityStateChanged` event emission (ADR-0006 event envelope: event_name, event_version, entity_schema_version, event_id, draft_id, occurred_at)
- [ ] Audit trail (who changed what, when, previous state) — generic across entity types
- [ ] Migrate material's approve flow (`mat applications` → `mat values`) onto `entity_drafts` → `entities` promotion
- [ ] Unit tests: state machine edge cases (invalid transitions, stale base_version, concurrent draft rejection)
- [ ] Deferred: post-promotion rollback model (explicitly out of scope for ADR-0005 — design separately if needed)

---

## Milestone 4 — Search & Distribution generalization

### Search
- [ ] Elasticsearch index template generation from registry config (material: Elasticsearch per ADR-0002; `pg_trgm` path stays implemented in Search Service for future lower-volume entity types, but material does not use it)
- [ ] Index sync pipeline (data service write → index update), async
- [ ] Vectorization/embedding pipeline (optional per entity type flag)
- [ ] Semantic retrieval query path (opt-in per entity type)
- [ ] Migrate material's ES index onto the generic index-template system
- [ ] Search relevance testing / benchmark against current material search

### Distribution
- [ ] Subscriber Registry (separate domain per ADR-0007): subscriber_id, display_name, owner_contact, status
- [ ] Subscription records: subscriber_id × entity_type, priority_tier, delivery_method, event_versions map, rate_limit_override, retry_policy, status
- [ ] Generic staging table (`distribution_record`: entity_type, payload, target subscriber, status)
- [ ] Replication queue (RabbitMQ, fed by the outbox relay's `EntityCommitted` publish, ADR-0008/0009 → staging table)
- [ ] Outbound priority queue dispatcher (business priority rules evaluated in code, not in broker; reads entity type's tier SLA + subscriber's subscription)
- [ ] Dual-version event emission per subscriber's `event_versions` map (ADR-0006) — dispatcher transforms committed entity into the shape each subscriber's declared version expects
- [ ] Push delivery path (RabbitMQ queue per subscriber, ADR-0008)
- [ ] Pull delivery path (subscriber-initiated fetch of individual missed records — for legacy downstream systems, or a subscriber catching up after downtime; no platform-triggered full resync, see §6)
- [ ] Rate limit / throttling per subscriber (top-bound / low-bound), backed by Redis, subscriber override support
- [ ] Retry + dead-letter handling for failed deliveries via RabbitMQ dead-letter exchanges (ADR-0008)
- [ ] Migrate material's 246 downstream subscribers into Subscriber Registry + subscriptions
- [ ] KEDA autoscaling config for replication/outbound consumer pods on RabbitMQ queue depth (cross-team: k8s admin support)
- [ ] Event version retirement process (governance, per ADR-0006) — track which subscriber is on which version, confirm migration before retiring an old version

---

## Milestone 5 — Gateway & MCP

- [ ] REST gateway (generic entity_type-parameterized routes)
- [ ] gRPC gateway (for dynamic client data structure requirement)
- [ ] Rate limiting middleware (Redis-backed, per identity)
- [ ] MCP server exposing dynamically generated tools per entity type (query_/create_/update_ per registered, AI-enabled entity type)
- [ ] Tool permission enforcement from registry's AI-access flags
- [ ] AuthN/authZ layer (service identity + human identity paths)
- [ ] API versioning strategy for gateway routes

---

## Milestone 6 — Second entity type onboarding (genericity test)

- [ ] Pick second entity type (vendor or customer) — confirm with stakeholders
- [ ] Register schema, workflow, index, and distribution config for it — config only, zero service code changes
- [ ] Onboard at least one real downstream subscriber for the new entity type
- [ ] Retrospective: document any place genericity broke down (leaky abstraction) and whether it needs a plugin/business-logic layer

---

## Cross-cutting / ongoing

- [ ] Infra: NAS → object storage migration (dual-write cutover plan)
- [ ] Observability: metrics + tracing across all services (queue depth, replication lag, index sync lag)
- [ ] Documentation: architecture doc, registry schema format spec, onboarding guide for adding a new entity type
- [ ] Security review: MCP tool exposure surface, rate limit bypass scenarios
- [ ] Disaster recovery: backup/restore procedure for registry config (losing this is losing the whole platform's behavior)
- [ ] Decide plugin/business-logic layer design (WASM vs dynamically loaded modules) — deferred, revisit after Milestone 6 retrospective

---

## Explicitly out of scope for now

- Entity-specific business logic beyond validation/workflow (deferred per prior discussion)
- Multi-region active-active writes (multi-region sync is outbound-only for now)
