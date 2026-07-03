# Registry schema format spec

This defines the shape of a single Entity Type Registry entry — the config document that makes a service generic for a given entity type. Every service in the platform reads entries in this format; none of them interpret entity semantics beyond it.

Examples below use JSON for illustration. The actual serialization (JSON vs a custom DSL) is an open decision — see ADR-0001.

## 1. Top-level entry

Every registry entry has five sections, matching the five things a generic service needs to know about an entity type:

```
entity_type: string              # unique key, e.g. "material", "vendor"
schema_version: integer          # incremented on any breaking field change
schema: { ... }                  # section 2
workflow: { ... }                # section 3
index: { ... }                   # section 4
distribution: { ... }            # section 5
ai_access: { ... }               # section 6
```

## 2. `schema` — field definitions

Describes the payload shape for this entity type. Data Service validates every write against this.

```json
{
  "business_key_field": "code",
  "fields": {
    "code": { "type": "string", "required": true, "max_length": 40 },
    "description": { "type": "string", "required": true, "max_length": 500 },
    "unit_of_measure": { "type": "string", "required": true, "enum": ["EA", "KG", "M", "L"] },
    "region": { "type": "string", "required": false },
    "attributes": { "type": "object", "required": false }
  }
}
```

`business_key_field` names which payload field Data Service copies into the envelope's `business_key` column on write (see `adr/0004-entity-envelope-model.md`). It must reference a `required` field of type `string` or `number`. This is the only place the envelope and payload are explicitly linked — every other envelope column (`workflow_state`, `schema_version`, `version`, timestamps) is platform-managed and never derived from payload.

Supported field types (v1): `string`, `number`, `boolean`, `enum`, `object` (nested, no schema enforcement inside — treated as opaque JSON), `array`.

Constraints supported per field: `required`, `max_length`, `min`/`max` (numeric), `enum`, `pattern` (regex, string only).

**Not supported in v1:** cross-field validation rules (e.g. "field A required only if field B = X"). If an entity type needs this, it's a signal for the plugin/business-logic layer discussion, not an extension to this spec.

## 3. `workflow` — approval state machine

Describes the states an entity instance can be in and the legal transitions between them.

```json
{
  "states": ["draft", "pending_approval", "rejected", "approved", "archived"],
  "initial_state": "draft",
  "transitions": [
    { "from": "draft", "to": "pending_approval", "action": "submit", "roles": ["author"] },
    { "from": "pending_approval", "to": "approved", "action": "approve", "roles": ["approver"] },
    { "from": "pending_approval", "to": "rejected", "action": "reject", "roles": ["approver"] },
    { "from": "approved", "to": "archived", "action": "archive", "roles": ["maintainer"] }
  ],
  "promotion_state": "approved",
  "distributable_states": ["approved"]
}
```

`promotion_state` — the single state whose transition promotes a record from the draft store into the Entity Store (see `adr/0005-workflow-persistence-model.md`). Before reaching this state, a record exists only as a draft; Data Service, Search Service, and Distribution Service never see it. States other than `promotion_state` (draft, pending_approval, rejected) are draft-only states and never appear as an entity record's `workflow_state`.

`distributable_states` — explicit list of entity-record states (post-promotion) eligible for distribution. Stated explicitly rather than inferred, so a state like `archived` can be deliberately included (downstream systems receive a deactivation signal) or excluded (downstream systems simply stop receiving updates) without relying on a guessed rule about terminal states.

Entity types that don't need approval (e.g. a simple lookup table) can omit `workflow` entirely — Workflow Service treats a missing definition as "no draft stage, every write promotes immediately, every state is distributable."

## 4. `index` — search configuration

```json
{
  "backend": "elasticsearch",
  "searchable_fields": ["code", "description", "attributes.*"],
  "embedding": {
    "enabled": false
  }
}
```

`backend` — `"elasticsearch"` or `"pg_trgm"`. See ADR-0002 for the decision criteria; this field is where that decision is expressed per entity type.

`embedding.enabled` — if `true`, Search Service runs the vectorization pipeline on write and exposes semantic retrieval for this entity type. Off by default; most entity types won't need it.

## 5. `distribution` — subscriber and delivery config

This section declares the entity type's SLA vocabulary and defaults — what priority tiers exist and what they guarantee, and what delivery methods are supported. It is not where individual subscribers register; a subscriber's own consumption details (which entity types, which tier, which event versions, delivery/retry config) live in the Subscriber Registry, a separate domain — see `adr/0007-subscriber-registry-ownership.md` for the full ownership split and subscription shape.

```json
{
  "priority_tiers": {
    "high": { "max_lag_seconds": 60 },
    "standard": { "max_lag_seconds": 3600 }
  },
  "rate_limit": {
    "requests_per_minute": 500
  },
  "delivery_methods": ["push", "pull"]
}
```

`priority_tiers` — named tiers a subscriber can be assigned to, each with a consistency SLA (`max_lag_seconds`) the outbound dispatcher must respect. This is where the "final consistency requirement" from the original design notes becomes an explicit, per-entity-type number instead of an implicit assumption. Individual subscribers select one of these tiers in their Subscriber Registry entry; this section only defines what the tiers mean.

## 6. `ai_access` — MCP exposure

```json
{
  "queryable": true,
  "writable": false,
  "approval_required_for_write": true
}
```

Gateway reads this to decide which MCP tools to generate for this entity type. `writable: false` means no `create_<entity_type>` / `update_<entity_type>` tool is exposed to the AI agent at all, regardless of the requester's role — this is a platform-level safety gate, not something workflow roles can override.

## 7. Versioning

- `schema_version` increments on any change to `schema.fields` that could break an existing stored payload (removing a required field, changing a type, tightening an enum).
- Non-breaking changes (adding an optional field, loosening a constraint) do not require a version bump but should still be recorded in the registry's change history.
- Data Service validates incoming writes against the *current* schema_version only. Existing stored payloads under an older schema_version are not retroactively validated — a migration task (out of scope for this spec) handles backfilling if a breaking change requires it.

## 8. Example: full entry for "material"

```json
{
  "entity_type": "material",
  "schema_version": 1,
  "schema": {
    "business_key_field": "code",
    "fields": {
      "code": { "type": "string", "required": true, "max_length": 40 },
      "description": { "type": "string", "required": true, "max_length": 500 },
      "unit_of_measure": { "type": "string", "required": true, "enum": ["EA", "KG", "M", "L"] },
      "region": { "type": "string", "required": false }
    }
  },
  "workflow": {
    "states": ["draft", "pending_approval", "rejected", "approved", "archived"],
    "initial_state": "draft",
    "transitions": [
      { "from": "draft", "to": "pending_approval", "action": "submit", "roles": ["author"] },
      { "from": "pending_approval", "to": "approved", "action": "approve", "roles": ["approver"] },
      { "from": "pending_approval", "to": "rejected", "action": "reject", "roles": ["approver"] },
      { "from": "approved", "to": "archived", "action": "archive", "roles": ["maintainer"] }
    ],
    "promotion_state": "approved",
    "distributable_states": ["approved"]
  },
  "index": {
    "backend": "elasticsearch",
    "searchable_fields": ["code", "description"],
    "embedding": { "enabled": false }
  },
  "distribution": {
    "priority_tiers": {
      "high": { "max_lag_seconds": 60 },
      "standard": { "max_lag_seconds": 3600 }
    },
    "rate_limit": { "requests_per_minute": 500 },
    "delivery_methods": ["push", "pull"]
  },
  "ai_access": {
    "queryable": true,
    "writable": false,
    "approval_required_for_write": true
  }
}
```

## 9. Open questions

- Should `schema.fields` support nested object validation (schema inside `attributes`), or stay opaque JSON as in v1? Opaque is simpler but pushes validation responsibility elsewhere if a second entity type needs it.
- Is JSON the right serialization, or does a custom DSL (more readable for non-engineers maintaining entity types) pay for itself? See ADR-0001.
- `distributable_states` needs to become explicit rather than inferred once a second entity type is registered with a workflow that doesn't end in a single terminal state.
