# ADR-0006: Event Contract Versioning

## Status

Accepted

## Context

ADR-0005 gave the platform's event stream concrete meaning for the first time: promotion emits `EntityCommitted`, and the draft lifecycle implies at least `DraftSubmitted` and `DraftRejected` alongside post-promotion transitions like `EntityArchived`. `ARCHITECTURE.md` has referred to "committed" and "state changed" events since the first draft of this design, but never defined their schema or how that schema changes over time.

This matters more here than in a typical internal event bus because Distribution Service fans these events out to up to 246 downstream subscribers per entity type (material's current count), most of which are systems this platform doesn't control the release schedule of. An unversioned event schema means any field addition, rename, or type change is a breaking change for every subscriber simultaneously, with no way to roll out gradually or let slow-moving consumers catch up on their own timeline.

## Decision

Every event the platform emits carries an explicit `event_name` and integer `event_version`, and is treated as an immutable contract once any subscriber depends on it. Concretely:

**Event envelope** (wraps every event, regardless of type):
```
event_name             string    -- e.g. "EntityCommitted"
event_version           integer   -- e.g. 1 (the shape of this event's payload)
entity_schema_version    integer   -- the entities.schema_version of the entity this event describes (ADR-0004)
event_id                UUID      -- unique per emission, for idempotency/dedup on the consumer side
entity_type              string
entity_id                UUID      -- references entities.id
draft_id                 UUID, nullable -- the entity_drafts record this event traces back to, when applicable (ADR-0005)
occurred_at              timestamp
payload                  object    -- shape defined by (event_name, event_version)
```

`event_version` and `entity_schema_version` are deliberately separate: the first describes how this *event* is shaped, the second describes which version of the *entity type's schema* the underlying record was validated against. `EntityCommitted` can stay at `event_version: 2` across many different `entity_schema_version` values as the material schema itself evolves — conflating the two would make it impossible to audit "which schema was this commit validated against" independent of the event format's own history.

**Core event types (v1):**
- `DraftSubmitted` — draft entered `pending_approval` (or equivalent)
- `DraftRejected` — draft entered `rejected`
- `EntityCommitted` — draft promoted into `entities` (ADR-0005's promotion event)
- `EntityStateChanged` — post-promotion `workflow_state` transition (e.g. `approved` → `archived`)

**Versioning rules:**
1. A breaking change (field removed, field renamed, field type changed, required field added, semantic meaning of an existing field changed) to an event's payload always produces a new `event_version`, published alongside the old one — never a silent mutation of an existing version.
2. A non-breaking change (optional field added, new enum value added to a field that consumers should already treat as open) may be added to the current version without a bump, but must be documented as such.
3. Distribution Service supports emitting a given logical event at more than one version simultaneously during a migration window — a subscriber declares which version it consumes (part of subscriber registration, see ADR-0007) and receives that version's shape.
4. An old event version is only retired after every subscriber still consuming it has confirmed migration or has been explicitly offboarded — not on a fixed timer. This is a governance rule, not something the platform can enforce mechanically at v1; it's a process commitment paired with the mechanism.
5. Event schemas live in the registry alongside entity type schemas (a new registry section, `events`, deferred to implementation — this ADR fixes the versioning rules, not the registry storage format for them).

## Alternatives considered

- **No versioning, single evolving schema** — simplest to build, but makes every schema change a coordinated flag-day across every subscriber. Rejected outright given the 246-subscriber count; a single missed subscriber means a silent data loss or a hard failure on their side with no platform-side warning.
- **Versioning at the API/gateway level only, not per-event** — would version the REST/gRPC contract but leave the async event stream (which is what most of the 246 subscribers actually consume, per the outbound queue design) unversioned. Rejected because the event stream, not the request/response API, is the actual integration surface for most downstream systems.
- **Consumer-driven contract testing instead of explicit versioning** — valuable as a complementary practice, but doesn't by itself solve the problem of a subscriber that can't yet handle a new shape; it only detects the break, doesn't provide a path around it. Adopted as a future addition, not a substitute for versioning.

## Consequences

- Distribution Service's dispatcher needs to be version-aware: it may need to transform a committed entity into more than one event version shape for the same emission, if subscribers on different versions both need to receive it.
- Every event type needs a documented, explicit schema before it ships, not an implicit "whatever Rust struct we serialize this run." This is a real authoring cost the platform previously wasn't paying.
- Registry gains a new domain of config (event schemas) alongside entity, workflow, index, and distribution config — reinforcing the earlier point (from prior review) that the registry needs internal domain separation even if it stays one physical store for now.
- The retirement rule (governance, not mechanism) means old event versions can accumulate operational cost if migrations stall — this needs an owner and a tracked list of "which subscriber is on which version," which doesn't exist yet and is a natural fit for the subscriber registry (ADR-0007).

## Revisit when

- The registry's `events` section needs its actual schema format designed — this ADR intentionally left that as an implementation task rather than deciding it here, to keep this ADR focused on the versioning policy itself. Note this is the third ADR in a row (0004, 0005, 0006) to point at registry domain separation as a real need rather than a hypothetical one — by the time `events` config exists, splitting the registry into schema/workflow/index/distribution/event domains (at minimum as internal module boundaries, per the earlier review) should probably happen rather than being deferred again.
- A real breaking change is needed for `EntityCommitted` (the most subscribed event) — that's the first live test of whether the dual-version emission and retirement process actually works end to end, and may surface gaps this ADR didn't anticipate.
