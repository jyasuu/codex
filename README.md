# Codex

> A Rust-native, entity-agnostic master data platform: one generic pipeline for materials, vendors, customers, and whatever entity type comes next.

## Origin

Codex generalizes a material (`mat`) master data system originally built for Pouchen's ERP stack. That system hardcoded material's schema, approval workflow, and downstream distribution list directly into services. Codex extracts those three things into configuration owned by an **Entity Type Registry**, so the same services can serve any entity type without a code change.

"Material" becomes the first registered entity type, not a special case baked into the platform.

## Core idea

- **Generic services, config-driven behavior.** Data, workflow, search, and distribution services never mention "material" in code. They look up schema, workflow definition, index mapping, and subscriber list from the registry at request time.
- **Schema-as-data.** Entities are stored as a typed envelope (id, entity_type, version, status, timestamps) wrapping a `serde_json::Value` payload, validated at the boundary against the registry's schema for that entity type. This avoids a rebuild/redeploy cycle every time a field is added to an entity type.
- **Registry is the only entity-aware layer.** All entity-specific knowledge — field definitions, approval states, index templates, AI-tool exposure — lives in the registry as data, not in service logic.

## Architecture overview

```
Client / AI agent
       |
    Gateway            (MCP + REST/gRPC, rate limiting)
       |
  +----+----+----+
  |         |         |
 Data    Workflow   Search       <- generic services, entity-agnostic
  |         |         |
  +----+----+----+
       |
  Distribution        (replication + outbound queues)
       |
 Downstream systems   (per entity type subscribers)

  Data / Workflow / Search all read config from:
       Entity Type Registry
       (schema, workflow, index config)
         |         |         |
      Material   Vendor   Customer   <- registered entity types (config, not code)
```

See the accompanying architecture diagram from design discussion for the full picture, including how this generalizes the original mat-specific pipeline (ES/Postgres search, replicate + outbound priority queues, KEDA-scaled consumers, Redis rate limiting).

## Status

**Design phase.** Architecture and data flow are defined; no code has been written yet. This repository currently exists to hold design docs and, once started, the Cargo workspace.

## Proposed tech stack

- **Language/runtime:** Rust, Tokio
- **Storage:** Postgres (JSONB payload + partitioning by entity type), object storage for large attachments (replacing NAS)
- **Search:** Elasticsearch per entity type (material is on Elasticsearch; `pg_trgm` remains available per registry entry for lower-volume entity types — ADR-0002)
- **Rate limiting:** Redis
- **Messaging:** RabbitMQ for both the internal committed/state-changed event bus and per-subscriber outbound delivery queues (ADR-0008), written via a transactional outbox (ADR-0009); consumers autoscaled via KEDA on queue depth
- **AI integration:** MCP server exposing tools generated dynamically from the registry
- **API surface:** gRPC (internal/service-to-service) + REST gateway (broad downstream compatibility)

## Proposed repository layout (Cargo workspace)

```
codex/
  crates/
    common/          # entity envelope types, error handling, shared traits
    registry/         # Entity Type Registry service + schema/workflow/index config
    gateway/          # MCP + REST/gRPC gateway, rate limiting, dynamic tool generation
    data-service/      # generic CRUD, delegates validation to registry
    workflow/          # generic approval state machine engine
    search/            # per-entity-type index management + optional embeddings
    distribution/       # replication queue, outbound queues, subscriber fan-out
```

## Roadmap

1. **Entity Type Registry MVP** — schema, workflow, and index config storage; register "material" as the first entity type, mapped 1:1 to its existing fields.
2. **Generic Data Service** — CRUD against the registry-driven schema, running material traffic.
3. **Generic Workflow Service** — approval state machine expressed as config, replacing material's hardcoded approve flow.
4. **Search & Distribution generalization** — per-entity-type ES index templates; replication/outbound queues driven by subscriber registration instead of a hardcoded downstream list.
5. **Second entity type onboarding** — register vendor or customer as a genericity test; any special-casing found here reveals where the abstraction leaks.

## Naming alternatives considered

- **Loom** — weaves distinct entity types through shared services into fan-out delivery.
- **Strata** — layered generic tiers, entity type as the variable layer.
- **Ledger** — plain, enterprise-appropriate; emphasizes system of record.

## Open questions

- Is there a concrete near-term second entity type (vendor, customer, BOM), or is genericity speculative for now?
- How much entity-specific *business logic* exists beyond validation and workflow states? If real logic exists, a plugin architecture (dynamically loaded modules or WASM) will need its own design pass — intentionally out of scope for this first cut.

## License

TBD
