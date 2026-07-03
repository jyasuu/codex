# ADR-0005: Workflow Persistence Model — Draft-first

## Status

Accepted

## Context

ADR-0004 defined the entity envelope, including a `workflow_state` column — but left open where a record physically lives before it's approved. This is not a minor implementation detail: it determines what Data Service, Search Service, and Distribution Service actually see, and by extension what "committed" means throughout `ARCHITECTURE.md`.

Two models were on the table:

**Write-first** — every create/update lands directly in the `entities` table immediately, starting in an early workflow state (e.g. `pending_approval`). Approval is just a state transition on an already-persisted record. Every consumer of `entities` (Search, Distribution, ad hoc queries) must filter on `workflow_state` to avoid surfacing unapproved data.

**Draft-first** — create/update lands in a separate staging store (`entity_drafts`). The record only enters the `entities` table at the moment it reaches the workflow's `promotion_state` (see the registry spec update accompanying this ADR). Before that, Search, Distribution, and any consumer reading `entities` structurally cannot see it — there's nothing to filter, because it isn't there.

This choice cascades into audit, versioning, rollback, search visibility, and distribution — all four are cited directly in `ARCHITECTURE.md` and the earlier reviews as depending on this decision.

It's also worth noting Draft-first isn't a new idea for this platform — it's a generalization of a pattern the original material system already used: `mat applications` (change requests, typed A/C/E/U/L/R) were always a separate staging concept from `mat values` (approved data), with promotion happening at approval. This ADR formalizes that pattern as a platform mechanism rather than reintroducing it as material-specific behavior.

## Decision

Use Draft-first. Concretely:

- A new `entity_drafts` table holds records in any pre-`promotion_state` workflow state (`draft`, `pending_approval`, `rejected`, per the entity type's registry definition). Shape: `id`, `entity_type`, `business_key`, `target_entity_id` (nullable — set when the draft is a change to an existing entity, null when creating a new one), `base_version` (nullable — the `entities.version` this draft was opened against, null for new-entity drafts), `schema_version`, `workflow_state`, `payload`, `submitted_by`, `created_at`, `updated_at`.
- Workflow Service operates on `entity_drafts` for every pre-promotion transition (submit, approve, reject, request changes). Rejected drafts stay in `entity_drafts` (or are deleted per retention policy) and never touch `entities`.
- When a draft's transition target matches the entity type's `promotion_state`, Workflow Service performs the promotion: check the draft's `base_version` against the current `entities.version` for `target_entity_id` (if the draft is a change to an existing entity) and reject promotion as stale if they don't match; validate payload against the current registry schema one final time; upsert into `entities`, setting `source_draft_id` to this draft's id, incrementing `entities.version`, and setting `entities.workflow_state` to the promotion state; then remove or archive the draft.
- Promotion is the point at which the "committed" event (per `ARCHITECTURE.md` §3) is emitted. Search and Distribution only ever consume committed entities — they have no code path that reads `entity_drafts`.
- After promotion, further transitions (e.g. `approved` → `archived`) happen directly on the `entities` record's `workflow_state` — they don't go back through the draft store, since the record is already committed and those transitions don't represent unreviewed changes to business data. Whether a post-promotion state is still distributable is governed by `distributable_states`, not by draft/entity-table location.

## Alternatives considered

- **Write-first** — simpler in one sense (one table, no promotion step), but pushes `workflow_state` filtering into every consumer of `entities`: Search Service must exclude non-distributable states from its index (or index them and filter at query time, leaking unapproved data if that filter is ever forgotten), Distribution Service must filter before fan-out, and any future ad hoc reporting query inherits the same obligation. The failure mode is a forgotten `WHERE workflow_state = 'approved'` clause somewhere down the line — a correctness bug that draft-first makes structurally impossible, because unapproved records simply aren't in the table consumers read from. Rejected primarily on this governance-safety basis, given the platform's 246-subscriber distribution surface amplifies the cost of that failure mode.
- **Write-first with a materialized "approved" view** — a middle ground where `entities` holds everything but a view or separate read-replica exposes only approved records. Rejected as unnecessary complexity: it recreates draft-first's separation (two things to read from) without draft-first's simplicity (no promotion step, no dual-write to reconcile).

## Consequences

- Two tables instead of one for entity storage (`entity_drafts` + `entities`), plus promotion logic in Workflow Service that must be transactional (schema validation, upsert, version increment, draft removal, event emission — this needs to succeed or fail atomically, not partially).
- Search and Distribution stay simpler: they read only from `entities` and never need workflow-state-aware filtering logic beyond `distributable_states`, since anything unapproved structurally doesn't reach them.
- Rollback of an approved entity to a prior revision is a new, distinct operation from "reject a draft" — rejecting only applies pre-promotion. Post-promotion rollback (e.g. reverting `entities` to `version: 26` after a bad `version: 27` promotion) is not designed by this ADR and needs its own follow-up if the platform requires it; flagged as a gap, not silently assumed away.
- Concurrent drafts against the same entity (two people editing material M123 at once) need a conflict resolution rule. This ADR adopts two layers: (1) Workflow Service rejects a new draft submission if an open draft already exists for the same `(entity_type, target_entity_id)`, and (2) as a backstop, `base_version` lets promotion detect and reject a stale draft — one opened against an `entities.version` that's since moved — even if the single-open-draft rule is bypassed or relaxed later. Confirm the exact rejection UX (silent block vs. surfaced conflict) during Milestone 3; the mechanism itself is decided here.
- Entity types that don't need approval (per the registry spec's "omit `workflow` entirely" case) skip `entity_drafts` altogether — Data Service writes straight to `entities`. Draft-first is the general case; it degrades cleanly to write-first behavior for entity types that opt out of workflow.

## Revisit when

- Post-promotion rollback becomes a concrete requirement — design it as its own ADR rather than retrofitting it here.
- A second entity type's approval process needs multiple sequential drafts against the same target entity (e.g. amendment chains) — the current `target_entity_id` model assumes one active draft at a time; that assumption should be tested against a real case before generalizing further.
