# ADR-0004: Entity Envelope Model

## Status

Accepted

## Context

ADR-0003 established a single generic `entities` table with an `id` / `entity_type` / `payload JSONB` shape. In practice, that shape is too thin: it conflates two categories of data that behave very differently.

- **Platform-controlled metadata** — identity, lifecycle state, and versioning information that Workflow Service, Search Service, and Distribution Service need to read on essentially every request, regardless of entity type: which business key this record is, what workflow state it's in, which schema version it was written against, when it last changed.
- **Business payload** — the entity-type-specific fields defined by that entity type's registry schema (material's `unit_of_measure`, vendor's `payment_terms`, etc.), which is exactly what ADR-0001 correctly keeps dynamic.

Storing the first category inside `payload` means every service that needs it has to reach into JSON (`payload->>'status'`, `payload->>'material_no'`) for data that isn't actually entity-specific — it's identical in shape across every entity type. That's both a performance cost (JSONB extraction instead of a column index) and a design smell: it makes platform metadata look optional and entity-owned when it's neither.

This does not reopen ADR-0001. Schema-as-data is still correct for business fields. The question here is narrower: what belongs in typed columns versus inside `payload`.

## Decision

Split every entity record into two layers:

**Envelope (platform-controlled, typed columns):**
```
id                UUID, primary key
entity_type       string
business_key      string           -- entity type's natural identifier, e.g. material code
workflow_state     string           -- current state per the entity type's workflow definition
schema_version     integer          -- which registry schema_version this payload was validated against
version            integer          -- optimistic concurrency counter
source_draft_id    UUID, nullable   -- the entity_drafts record this version was promoted from (ADR-0005)
created_at         timestamp
updated_at         timestamp
payload            JSONB
```

**Payload (entity-controlled, dynamic):** business fields as defined by the entity type's registry schema — everything currently described in `registry-schema-spec.md` section 2.

`business_key` and `workflow_state` are indexed columns. `(entity_type, business_key)` becomes the primary lookup path for the ~90% of queries that are "get me this entity type's record by its natural key," rather than a JSONB path expression.

`workflow_state` in the envelope is the source of truth; Workflow Service is the only writer of this column. Data Service, Search Service, and Distribution Service read it directly instead of parsing it out of payload.

**`schema_version` vs `version` — these track different things and must not be conflated:**

- `schema_version` is the registry's schema version this record's payload was validated against (e.g. "material schema definition v3"). It changes only when the entity *type's* schema changes in the registry — it says nothing about this individual record's history.
- `version` is this individual record's revision counter (e.g. "material M123, revision 27"). It increments on every write to that record and is what optimistic concurrency checks against — it says nothing about the schema.

A record can be at `version: 27` while still validated against `schema_version: 1`, if the material schema hasn't changed since the type was registered. Conversely, two different records of the same entity type can legitimately sit at different `schema_version` values if one was last written before a schema change and hasn't been touched since — Data Service does not retroactively revalidate untouched records (see `registry-schema-spec.md` §7).

## Alternatives considered

- **Pure JSONB (payload-only), current ADR-0003 wording** — simplest possible schema, but pushes every service into JSON path queries for data that's structurally identical across all entity types. Rejected because it doesn't actually buy flexibility — `workflow_state` isn't dynamic per entity type, it's dynamic *in value* but fixed *in shape*, which is exactly what a typed column is for.
- **Fully typed columns per entity type (extending into codegen)** — would solve the query performance concern completely but reopens ADR-0001's rejected codegen path for business fields. Rejected for the same reasons as ADR-0001.

## Consequences

- `(entity_type, business_key)` gets a standard B-tree index, giving fast natural-key lookups without relying on GIN/JSONB extraction for the most common query shape.
- Workflow Service, Search Service, and Distribution Service can filter and join on `workflow_state` and `schema_version` directly, rather than every consumer independently parsing them out of payload — this also closes a consistency risk where different services might read a stale or differently-formatted copy of state from inside JSON.
- `business_key` uniqueness (per entity_type) needs to be enforced at the application layer or via a partial unique index — this wasn't a concern under pure JSONB but matters once it's a first-class column other logic depends on.
- Registry schema definitions (`registry-schema-spec.md`) need a small addition: each entity type must declare which of its payload fields maps to `business_key` at write time, so Data Service knows where to copy it from.
- This does not solve cross-entity referential integrity (still an application-layer concern per ADR-0003) — it only makes each entity's *own* identity and lifecycle queryable without JSON parsing.

## Revisit when

If a second entity type's workflow needs lifecycle metadata beyond `workflow_state` (e.g. a distinct "distribution eligibility" flag that isn't the same as workflow state), extend the envelope rather than routing it through payload — the same reasoning that put `workflow_state` in the envelope applies to any other cross-cutting, structurally-identical field a future entity type surfaces.
