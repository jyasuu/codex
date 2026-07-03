# ADR-0001: Schema-as-data over codegen for entity payloads

## Status

Accepted

## Context

Codex must support multiple entity types with different field shapes in a statically typed language (Rust). Two broad approaches exist:

1. **Schema-as-data** — entities stored and passed around as a typed envelope wrapping a `serde_json::Value` payload, validated at the boundary against the registry's schema for that entity type.
2. **Codegen** — generate Rust structs from each entity type's registry definition at build or deploy time, giving full compile-time type safety for entity fields.

The choice affects every generic service, since it determines what "an entity" looks like in code throughout the platform.

## Decision

Use schema-as-data for the core platform (Data Service, Workflow Service, Search Service, Distribution Service, Gateway). Entity payloads are `serde_json::Value`, validated against the registry schema at the boundary, not represented as per-entity-type Rust structs.

## Alternatives considered

- **Codegen** — stronger type safety inside services, but requires a build/deploy step every time an entity type's schema changes. Given the platform's whole premise is that adding or changing an entity type should not require a code change, codegen works against the core goal. It's a better fit for a system with few, slowly-changing entity types — not this one.
- **Hybrid (schema-as-data core, codegen for specific entity types with heavy logic)** — deferred. Revisit if the Milestone 6 second-entity-type onboarding surfaces a real need for entity-specific typed logic; see the open plugin/business-logic layer question in `ARCHITECTURE.md`.

## Consequences

- Onboarding a new entity type is a registry write, not a deploy — matches the stated success criterion for genericity.
- Field-level typos or type errors in entity payloads are caught by schema validation at runtime, not by the Rust compiler. Validation code and its test coverage carry more weight than they would under codegen.
- Internal service code works with `serde_json::Value`, which is less ergonomic than typed structs — expect more explicit validation and error-handling code in Data Service than a typed approach would need.
- This mirrors the event-routing pattern already used in the `pgx` project, where heterogeneous payloads are handled generically rather than per-sink-type.

## Revisit when

An entity type is onboarded (Milestone 6) that needs computed fields or cross-field business rules the schema spec can't express — that's the point to decide whether a plugin layer supplements schema-as-data rather than replacing it.
