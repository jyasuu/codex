# Architecture

This document goes deeper than the README's overview: component responsibilities, data flow, deployment shape, and the non-functional requirements the design has to satisfy. It assumes you've read the README first.

## 1. Design goal

One platform, N entity types. Material is the first entity type it serves; the platform itself must not know what "material" means. Anything entity-specific is data in the registry, never code in a service.

The test of success isn't "does material work" — material already works today in the legacy system. The test is: **can a second entity type go live with zero changes to `data-service`, `workflow`, `search`, `distribution`, or `gateway`** — only a new registry entry and a subscriber registration.

## 2. Components

### 2.1 Entity Type Registry

The only entity-aware component. Owns:
- Field schema per entity type (types, validation, versioning)
- Workflow definition (states, transitions, permitted roles)
- Index/search definition (which fields are searchable, embedding on/off)
- Distribution profile (who subscribes, delivery method, priority tier)
- AI-access flags (which entity types and operations an AI agent may reach via MCP)

Read-heavy, write-rare. Every other service treats it as a config source, cached locally with invalidation on registry change (see §5).

See `registry-schema-spec.md` for the exact format.

### 2.2 Gateway

Single entry point for clients and the AI agent. Responsibilities:
- Route by entity_type + operation to the right generic service
- Rate limiting (Redis-backed, per identity)
- Protocol translation (REST and gRPC both terminate here)
- MCP tool generation: on registry change, regenerate the set of exposed MCP tools (`query_<entity_type>`, `create_<entity_type>`, etc.) filtered by AI-access flags

The gateway does not touch entity payloads beyond routing — no entity-specific logic here either.

### 2.3 Data Service

Generic CRUD over the entity envelope. On every write:
1. Look up schema for `entity_type` from registry cache
2. Validate payload against schema
3. Persist envelope + payload to Postgres
4. Emit a "committed" event (consumed by search sync and replication)

No entity-type branching in code — validation and persistence are schema-driven.

### 2.4 Workflow Service

Generic state machine engine, operating on the draft store (`entity_drafts`), not on committed entities directly — see ADR-0005 for the full model. Given an entity_type's workflow definition (states + transitions + role checks), it:
- Accepts submit/approve/reject/request-changes actions against a draft
- Validates the requested transition is legal from the draft's current state
- Writes an audit trail entry per transition
- On a transition into the entity type's `promotion_state`, performs promotion: final schema validation, upsert into `entities`, version increment, and emits the "committed" event — this is the only point at which a record becomes visible to Search and Distribution
- Post-promotion transitions (e.g. `approved` → `archived`) operate directly on the `entities` record's `workflow_state`, without a further draft cycle

Replaces material's hardcoded approve flow and A/C/E/U/L/R change-type enum — those become one entity type's workflow definition, not platform code. This is also a direct generalization of the original mat system's `mat applications` (drafts/change requests) → `mat values` (approved) split.

### 2.5 Search Service

Per-entity-type index management:
- On registry change, generate/update the index template for that entity type (ES mapping, or route to `pg_trgm` if the entity type's registry entry says so — see ADR-0002)
- Consumes "committed" events to keep the index in sync (async, not inline with writes)
- Optional embedding pipeline for entity types with semantic retrieval enabled

### 2.6 Distribution Service

Generic fan-out:
- Consumes "committed" and "state changed" events into a staging table (`distribution_record`)
- Priority-queue dispatcher, evaluated in code against the entity type's distribution profile (priority is business logic, not broker config — see prior discussion)
- Delivery paths: push (webhook/queue per subscriber) and pull (subscriber-initiated fetch, for legacy downstream systems that can't accept pushes)
- Retry + dead-letter handling per subscriber

## 3. Data flow (write path)

```
Client/AI agent -> Gateway -> Workflow Service (entity_drafts)
                                       |
                          submit / approve / reject
                                       |
                        transition into promotion_state?
                                       |
                                promote to Data Service
                                       |
                       validate against registry schema (final check)
                                       |
                            persist envelope+payload to entities
                                       |
                          emit "committed" event
                             /         \
                     Search Service   Distribution Service
                    (index sync)      (stage -> queue -> subscribers)
```

Every create/update enters as a draft (`entity_drafts`) and is only promoted into `entities` — and only then visible to Search and Distribution — when its workflow reaches the entity type's `promotion_state`. See ADR-0005 for the full draft/promotion model, and the registry spec's `distributable_states` for which post-promotion states remain eligible for distribution (e.g. an archived entity may or may not still be sent downstream, depending on the entity type's config).

Entity types with no `workflow` definition skip the draft stage entirely — Data Service writes straight to `entities`, and every state is distributable by default.

## 4. Storage shape

- **Draft store:** one generic Postgres table (`entity_drafts`: id, entity_type, business_key, target_entity_id, base_version, schema_version, workflow_state, payload JSONB, submitted_by, timestamps) — holds every pre-promotion record. `base_version` records the `entities.version` the draft was opened against, letting promotion detect a stale draft. See ADR-0005.
- **Primary store:** one generic Postgres table (`entities`: id, entity_type, business_key, workflow_state, schema_version, version, source_draft_id, payload JSONB, timestamps — see ADR-0004 for the envelope/payload split), GIN-indexed on payload, B-tree indexed on `(entity_type, business_key)`, partitioned by entity_type (+ secondary key per entity type's volume profile).
- **Distribution staging:** one generic table (`distribution_record`: entity_type, payload, target_subscriber, status, attempts), JSON upsert rather than per-entity-type tables (see ADR-0003).
- **Search:** one ES index per entity type (or `pg_trgm` for entity types that don't need it), template generated from the registry, not hand-maintained.
- **Large attachments:** object storage, not NAS (durability + pay-as-usage; migration is a separate cross-cutting task).

## 5. Registry propagation

Services don't hit the registry on every request — that would put registry latency on the hot path for every read/write, across every entity type. Instead:

- Each service caches registry entries in memory, keyed by entity_type + schema_version.
- Registry writes publish an invalidation event; subscribed services refresh the affected entity_type's cache entry.
- Services that are down during a registry change pick up the new version on next cache refresh or restart — this means a schema change is eventually consistent across the platform, not instantaneous. Acceptable given registry writes are rare (an entity type onboarding or a field addition, not routine traffic).

## 6. Non-functional requirements (carried over from the original mat system)

These applied to material specifically; as generic requirements they now apply per entity type, sized by that entity type's registry-declared volume profile:

- **Search:** full-text + structural search at scale (material: 26M rows / 13GB) without dominating disk IO/CPU — decision on ES vs `pg_trgm` is per entity type (ADR-0002).
- **Rate limiting:** top-bound and low-bound throttling per consumer identity, particularly for SAP-style high-volume consumers — Redis-backed, generic across entity types.
- **Outbound scalability:** fan-out to N subscribers without a broadcast-storm — confirm actual sync cadence (full vs incremental) before assuming broadcast is safe at any entity type's subscriber count. The original bandwidth estimate for material (20,000 records × 2KB × 246 systems) works out to roughly 9.8 GB per full broadcast — at 30 Mb/s intranet bandwidth that's tens of minutes, which rules out full broadcast on every write and pushes the design toward incremental delivery per subscriber.
- **Autoscaling:** consumer pods (replication, outbound) scale with queue depth via KEDA, not fixed replica counts.
- **Consistency:** each entity type's registry entry declares its consistency SLA (how long a downstream system may lag behind a committed write) so the rate limiter and outbound queue can be tuned without silently violating it.

## 7. Deployment topology (proposed)

- Registry, Gateway, Data Service, Workflow Service, Search Service, Distribution Service each deploy independently (separate binaries/containers), communicating over gRPC internally.
- Postgres (primary store), Redis (rate limiting + cache invalidation pub/sub), message broker (event bus for committed/state-changed events), Elasticsearch (optional, per entity type) as stateful dependencies.
- KEDA-managed autoscaling on Search sync consumers and Distribution outbound consumers, since those are the queue-depth-driven components.

## 8. What's deliberately not generic (yet)

- **Entity-specific business logic** beyond validation and workflow states (e.g., computed fields, cross-field business rules) has no home in this design yet. If Milestone 6's second entity type surfaces a real need for this, it becomes a plugin architecture decision (WASM vs dynamically loaded modules) — tracked as an open question, not designed here.
- **Multi-region active-active writes.** Multi-region sync in this design is outbound delivery only; it does not attempt multi-region write consistency.

## 9. Design status

The core architectural decision set is closed as of ADR-0007. Together, ADR-0001 through ADR-0007 answer, in order: how an entity is represented, where it's stored, what its envelope looks like, how it enters the system, how it propagates as events, and how external systems consume it — each ADR's decision depends on and is consistent with the ones before it.

| ADR | Decision |
|---|---|
| 0001 | Schema-as-data over codegen |
| 0002 | Search backend chosen per entity type |
| 0003 | Single generic table, JSON upsert over per-entity-type tables |
| 0004 | Entity envelope — platform metadata vs. business payload |
| 0005 | Draft-first workflow persistence |
| 0006 | Event contract versioning |
| 0007 | Subscriber Registry ownership, separate from Entity Registry |

From here, the remaining risk is implementation complexity, not architectural direction — particularly the promotion transaction (ADR-0005), dual-version event emission (ADR-0006), and the real cost of migrating material's 246 existing subscribers onto the Subscriber Registry (ADR-0007).

## 10. Deferred — phase 2 concerns

These surfaced repeatedly across the ADR sequence but were deliberately not designed yet, either because no concrete case exists to design against, or because doing so now would be speculative. Each has a natural trigger for when it should be picked up:

- **Registry domain separation** (schema/workflow/index/distribution/event as distinct internal namespaces) — flagged three times across ADR-0004, 0005, and 0006. Should likely be addressed as soon as the `events` registry section (ADR-0006) is implemented, rather than deferred a fourth time.
- **Event Registry model** — where and how event schemas (ADR-0006) are actually stored and versioned in the registry. Scoped out of ADR-0006 intentionally.
- **Authorization model** — role/permission enforcement beyond workflow transition roles (ADR-0005) hasn't been designed as a platform-wide concern.
- **Cross-entity references** — no relational integrity across entity types (ADR-0003's accepted limitation). Revisit at Milestone 6 if the second entity type has a real relationship to an existing one.
- **Post-promotion rollback** — reverting an already-committed entity to a prior revision (ADR-0005's explicitly undesigned gap).
- **Observability model** — metrics/tracing conventions across services, listed in `todo.md` cross-cutting tasks but not architecturally designed.
- **Multi-region strategy** — this design covers outbound distribution across regions, not multi-region active-active writes (§8, "what's deliberately not generic").
