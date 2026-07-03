# ADR-0007: Subscriber Registry Ownership

## Status

Accepted

## Context

`registry-schema-spec.md` currently defines a `distribution` section inside each entity type's registry entry — priority tiers, rate limits, delivery methods — as a *default profile*, with actual subscribers "registering separately" per `todo.md` Milestone 4. That separation was named early but never formally designed: there's no defined shape for what a subscriber registration actually contains, who owns its lifecycle, or how it relates to the entity type's registry entry.

ADR-0006 made this urgent rather than theoretical: a subscriber must now declare which `event_version` it consumes per event type. That's subscriber-specific state that changes on the subscriber's timeline (when they migrate), not the entity type's timeline (when the schema changes) — it cannot live inside the entity type's registry entry without conflating two different lifecycles, which is exactly the concern the earlier review raised about the Entity Registry becoming a "God Service."

This is also, concretely, the mechanism by which material's 246 existing downstream systems get modeled in Codex — this ADR is what Milestone 4 builds against.

## Decision

Subscriber registration is a separate registry domain from entity type definitions, with its own lifecycle. Concretely:

**Subscriber Registry entry** (one per subscriber):
```
subscriber_id         string    -- stable identifier, e.g. "sap-prod", "mes-taiwan"
display_name           string
owner_contact           string    -- who to reach when this subscriber needs migration/support
status                  string    -- active | suspended | offboarding | offboarded
```

**Subscription** (one per subscriber × entity_type pair a subscriber consumes):
```
subscriber_id
entity_type
priority_tier           string    -- references one of the entity type's declared tiers (registry §5)
delivery_method          string    -- push (webhook/queue) | pull
event_versions           map       -- { "EntityCommitted": 2, "EntityStateChanged": 1, ... } per event type
rate_limit_override       object, nullable -- overrides the entity type's default if this subscriber needs a different bound
retry_policy              object    -- max attempts, backoff strategy
status                    string    -- active | paused | migrating
```

**Ownership split:**
- Entity Registry (per entity type) continues to own: what priority tiers *exist* and their SLA (`max_lag_seconds`), what delivery methods are *supported*, default rate limit — this is vocabulary and defaults, not subscriber-specific state.
- Subscriber Registry owns: which subscribers exist, which entity types each one consumes, which tier and event version each one is on, and delivery/retry configuration specific to that subscriber.

Distribution Service reads both at dispatch time: the entity type's tier definition for the SLA, and the subscriber's subscription for delivery method, event version, and rate limit.

## Alternatives considered

- **Distribution profile stays inside Entity Registry, subscribers as a flat list** (the original registry spec's implicit model) — simplest to implement first, but conflates entity type lifecycle with subscriber lifecycle. A subscriber's migration to a new event version, or a new subscriber onboarding, would require touching the entity type's registry entry — exactly the coupling the ADR-0006 review flagged as a problem waiting to surface. Rejected.
- **Subscriber Registry owns everything, including SLA tier definitions** — the reviewer's more aggressive framing (move distribution entirely out of Entity Registry). Rejected in favor of the split: SLA tiers describe what an entity type's data freshness *can* support, which is a property of the entity type (how it's produced, how often it changes), not of any individual subscriber. Keeping tier vocabulary with the entity type avoids every subscriber having to redefine what "high priority" means for material specifically.

## Consequences

- Onboarding a new subscriber to an existing entity type is a Subscriber Registry write only — no Entity Registry change, matching the platform's core genericity goal extended to the distribution side.
- Distribution Service's dispatch logic now joins two registry domains at read time (entity type tier definitions + subscriber subscription config) instead of reading one flat structure — more moving parts, but each part changes on its own natural cadence instead of forcing unrelated changes to collide.
- Migrating material's 246 existing downstream systems (Milestone 4) becomes a bulk Subscriber Registry population task with a defined target shape, rather than an open-ended "figure out the model while migrating" problem.
- `event_versions` per subscriber means Distribution Service must track, per subscriber, which version of each event type to emit — this is the direct implementation of ADR-0006's dual-version-emission requirement, and gives it a concrete home.
- A subscriber's `status` lifecycle (active → paused → migrating → offboarding → offboarded) needs its own state transitions and who's authorized to trigger them — not fully specified here; treat as an implementation detail for Milestone 4, consistent with how this ADR series has handled similarly-scoped gaps elsewhere.

## Revisit when

- A subscriber needs per-field filtering (e.g. only wants a subset of a material's payload fields, not the full record) — this ADR assumes a subscriber receives the full committed payload; partial/filtered delivery is a materially different design and should get its own ADR if it becomes a real requirement.
- Milestone 4's actual migration of material's 246 subscribers surfaces a subscription shape this ADR didn't anticipate — expected, given this is the first time the model meets real data; amend rather than treating the shape above as final. A likely candidate: a `consumer_capabilities` block (ordering, batching, replay support) alongside `event_versions`, once Distribution Service's dispatcher needs to know not just *which* version a subscriber wants but *how* it can receive it. Not designed now — flagged because it's a predictable near-term addition, not a hypothetical one.
