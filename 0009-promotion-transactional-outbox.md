# ADR-0009: Transactional outbox for promotion event emission

## Status

Accepted

## Context

ADR-0005's promotion transaction — base_version staleness check, final schema validation, upsert into `entities`, draft removal, and emitting `EntityCommitted` — is described as needing to be atomic, and `ARCHITECTURE.md` §9 names it directly as the platform's top implementation risk. The Postgres writes (upsert into `entities`, draft removal) and the event emission to RabbitMQ (ADR-0008) cannot be one transaction — Postgres commit and broker publish are two separate systems. Publishing before the Postgres commit risks announcing a promotion that then fails to commit; committing before publishing risks a committed record that Search and Distribution never learn about if the process crashes between the two steps. Either ordering has a window where the two systems disagree.

## Decision

Use the transactional outbox pattern for promotion. The promotion transaction writes the `entities` upsert, the draft removal, and an `EntityCommitted` outbox row into the same Postgres transaction. A separate outbox relay process reads unpublished outbox rows and publishes them to RabbitMQ (ADR-0008), marking each row published only after broker confirmation.

Concretely:
- A generic `outbox_events` table (id, event_name, event_version, entity_type, entity_id, draft_id, payload, occurred_at, published_at nullable) is written in the same transaction as any state change that must emit an event — not just promotion, but also `DraftSubmitted`, `DraftRejected`, and `EntityStateChanged` (ADR-0006), so the same mechanism covers every event type rather than special-casing promotion.
- The outbox relay polls (or uses `LISTEN`/`NOTIFY` for low-latency wakeup) for rows where `published_at IS NULL`, publishes to RabbitMQ, and sets `published_at` on broker confirmation — at-least-once delivery, with `event_id` (ADR-0006's event envelope) giving consumers the idempotency key to dedupe.
- This directly reuses the transactional outbox pattern already implemented for AnvilNotify's outbox worker (adaptive poll backoff, stale lock reaper) rather than designing the mechanism from scratch.

## Alternatives considered

- **Publish-then-commit** — publish `EntityCommitted` to RabbitMQ, then commit the Postgres transaction. Rejected: if the Postgres commit fails or the process crashes after publish but before commit, subscribers receive an event for a promotion that never actually happened.
- **Commit-then-publish (no outbox)** — commit the Postgres transaction, then publish directly to RabbitMQ. Rejected: if the process crashes between commit and publish, the promotion is durably committed but no event is ever emitted — Search and Distribution silently never learn about it, with no retry mechanism, which is worse than a duplicate.
- **Change Data Capture (CDC) off `entities`/`entity_drafts`** (e.g. Debezium-style WAL tailing) — avoids a bespoke outbox table and relay process by deriving events directly from the write-ahead log. Rejected for now: it adds an external CDC component and ties event shape tightly to table structure (harder to keep `EntityCommitted`'s payload shape independent from `entities`' column shape, which ADR-0006's versioning model depends on). Reasonable to revisit if the outbox relay itself becomes an operational bottleneck.

## Consequences

- Every event-emitting transition (not just promotion) now has two writes to keep synchronized in review: the state-changing write and the outbox row — this is a pattern to enforce consistently (likely via a shared helper in `common`) rather than hand-rolled per call site, to avoid a call site that forgets the outbox insert.
- The outbox relay is a new long-running component (or a task within Workflow/Data Service) with its own failure mode to design for: stale/crashed relay instances holding a lock on rows they never published. AnvilNotify's stale lock reaper pattern is the direct precedent to reuse here.
- At-least-once delivery means every consumer (Search Service, Distribution Service, and downstream subscribers via ADR-0007) must dedupe on `event_id` — this was already implied by ADR-0006's event envelope including `event_id` "for idempotency/dedup on the consumer side," so this ADR doesn't introduce a new consumer obligation, it just makes concrete why that field exists.
- `outbox_events` adds write volume to the same Postgres instance as `entities`/`entity_drafts` — at material's scale this needs the same partitioning/retention discipline (old published rows archived or dropped on a schedule) as the primary tables, not left to grow unbounded.

## Revisit when

The outbox relay's poll/notify latency becomes a measurable bottleneck against a priority tier's `max_lag_seconds` SLA (registry §5) — at that point, CDC-based emission becomes worth reconsidering as a lower-latency alternative to polling-based relay.
