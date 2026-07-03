# ADR-0003: Single generic table with JSON upsert, not per-entity-type tables

## Status

Accepted

## Context

Storing entities generically across many entity types requires a decision on table shape: one physical table per entity type (mirroring how a traditional ERP would model `material`, `vendor`, `customer` as separate relational tables), or one generic table shared across all entity types, storing the type-specific payload as JSON.

This decision extends a conclusion already reached for the original mat system's `dc mat` staging table — JSON upsert there was judged simpler and less race-condition-prone than multi-table delete-insert — generalized to the primary entity store itself.

## Decision

Use a single generic table (`entities`: id, entity_type, payload JSONB, version, status, timestamps), partitioned by entity_type, with JSON upsert as the write pattern. Do not create a physical table per entity type.

The same pattern applies to the distribution staging table (`distribution_record`).

## Alternatives considered

- **One table per entity type** — gives natural relational structure and easy SQL querying per entity type, and would let the database enforce field types directly. Rejected because it reintroduces exactly what schema-as-data (ADR-0001) is avoiding: a schema migration (`ALTER TABLE`) every time an entity type's fields change, and a new table + migration for every new entity type — both are deploy-coupled operations the platform's core goal is trying to eliminate.
- **Multi-table delete-insert for updates** — considered and rejected for the same reason it was rejected for `dc mat` originally: higher risk of race conditions under concurrent writes, more complex replication logic, no clear benefit over upsert for this access pattern.

## Consequences

- Losing native relational structure means no foreign-key enforcement, no per-field SQL types, and JSONB querying (GIN index) instead of ordinary column indexes. Query performance for complex filters depends on GIN index tuning rather than being free from table structure.
- Partitioning by entity_type is the mechanism that keeps this from becoming one giant unpartitioned table as multiple entity types accumulate volume — this is not optional if the platform is going to hold more than one high-volume entity type.
- This table shape directly enables ADR-0001 (schema-as-data): the storage layer and the service layer make the same trade-off in the same direction, so they're not fighting each other.
- Cross-entity referential integrity (e.g. a material payload referencing a vendor by id) must be enforced by application services, not by database foreign-key constraints — there is no `FOREIGN KEY` mechanism across a generic JSONB table. Until a second, related entity type exists, this is an accepted limitation rather than a designed-around one; see "Revisit when."
- Identity and lifecycle metadata that services query on every request (business key, workflow state, schema version) should not live solely inside `payload` — see ADR-0004, which refines this table's column shape without changing the core decision made here.

## Revisit when

- A specific entity type's query patterns genuinely require relational joins or field-level SQL constraints that JSONB + GIN can't support efficiently — at that point, that entity type (not the whole platform) may warrant a dedicated table as an exception, decided case by case.
- Milestone 6 (second entity type onboarding): if the new entity type has a real relationship to an existing one (e.g. material referencing vendor), design a cross-entity reference validation mechanism then, against a concrete case rather than a hypothetical one.
