# ADR-0002: Search backend is chosen per entity type, not platform-wide

## Status

Accepted — including material's specific backend choice (Elasticsearch), resolved below.

## Context

Material's search requirement — 26M rows, 13GB, full-text plus complex structural search, with pressure to reduce search disk IO/CPU load — was originally scoped as an Elasticsearch decision. But as a generic platform, not every future entity type will have material's volume or query complexity. Forcing every entity type onto Elasticsearch adds operational cost (a second data store, a sync pipeline) that may not be justified for a small lookup-style entity type; forcing everything onto Postgres (`pg_trgm`) may not hold up for material's actual query patterns.

## Decision

The registry's `index.backend` field (see `registry-schema-spec.md`, section 4) makes search backend a per-entity-type choice, not a platform-wide one. Search Service supports both `elasticsearch` and `pg_trgm` and routes per entity type accordingly.

Material's own `index.backend` is set to `elasticsearch`. Material's requirement — full-text plus complex structural search at 26M rows / 13GB, with an existing production query pattern the legacy system already serves on Elasticsearch — is exactly the case ADR-0002's "Alternatives considered" section flags as the risk of committing to `pg_trgm` platform-wide: untested against material's actual query complexity at this volume. Rather than spend a prototyping cycle validating `pg_trgm` against a workload Elasticsearch is already known to serve correctly in production today, material stays on Elasticsearch and the mechanism (both backends supported, chosen per entity type) remains available for a future entity type whose volume or query shape doesn't justify a second data store.

## Alternatives considered

- **Elasticsearch platform-wide** — simplest to reason about, one search mechanism everywhere. Rejected as the default because it forces every future entity type into a second data store and a sync pipeline even when unnecessary.
- **`pg_trgm` platform-wide** — avoids a second data store entirely. Rejected as the default because it's untested against material's actual query complexity at 26M rows; committing to it platform-wide before validating against material's real patterns risks a costly reversal later.

## Consequences

- Search Service must support two backends, which is more implementation and operational surface than a single-backend design.
- Each entity type's registry entry carries the responsibility of declaring the right backend — a wrong choice at registration time means either wasted ES infrastructure or a slow `pg_trgm` fallback discovered late.
- Index template generation logic (from the registry's `index` section) needs to branch on backend, which is the one place Search Service isn't fully backend-agnostic internally, even though it stays entity-type-agnostic.

## Revisit when

A future entity type's volume and query pattern don't justify Elasticsearch's operational cost (a second data store, a sync pipeline) — that's the case to prototype `pg_trgm` against, using material's Elasticsearch entry as the reference point for what a "needs ES" workload looks like versus one that doesn't.
