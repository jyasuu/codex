# ADR-0002: Search backend is chosen per entity type, not platform-wide

## Status

Deferred (for material specifically) / Accepted (as a platform mechanism)

## Context

Material's search requirement — 26M rows, 13GB, full-text plus complex structural search, with pressure to reduce search disk IO/CPU load — was originally scoped as an Elasticsearch decision. But as a generic platform, not every future entity type will have material's volume or query complexity. Forcing every entity type onto Elasticsearch adds operational cost (a second data store, a sync pipeline) that may not be justified for a small lookup-style entity type; forcing everything onto Postgres (`pg_trgm`) may not hold up for material's actual query patterns.

## Decision

The registry's `index.backend` field (see `registry-schema-spec.md`, section 4) makes search backend a per-entity-type choice, not a platform-wide one. Search Service supports both `elasticsearch` and `pg_trgm` and routes per entity type accordingly.

The specific choice for material itself is deferred until both are prototyped against real material query logs (see "Revisit when").

## Alternatives considered

- **Elasticsearch platform-wide** — simplest to reason about, one search mechanism everywhere. Rejected as the default because it forces every future entity type into a second data store and a sync pipeline even when unnecessary.
- **`pg_trgm` platform-wide** — avoids a second data store entirely. Rejected as the default because it's untested against material's actual query complexity at 26M rows; committing to it platform-wide before validating against material's real patterns risks a costly reversal later.

## Consequences

- Search Service must support two backends, which is more implementation and operational surface than a single-backend design.
- Each entity type's registry entry carries the responsibility of declaring the right backend — a wrong choice at registration time means either wasted ES infrastructure or a slow `pg_trgm` fallback discovered late.
- Index template generation logic (from the registry's `index` section) needs to branch on backend, which is the one place Search Service isn't fully backend-agnostic internally, even though it stays entity-type-agnostic.

## Revisit when

Material's specific backend choice is unresolved — prototype both `elasticsearch` and `pg_trgm` (with GIN indexing) against real material query logs before Milestone 4 locks in material's registry entry. Whichever wins becomes the reference example other entity types compare against when choosing their own `index.backend`.
