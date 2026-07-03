# ADR-0008: Message broker — RabbitMQ

## Status

Accepted

## Context

`ARCHITECTURE.md` §7 lists a message broker as a stateful dependency (event bus for committed/state-changed events) without naming one, and `todo.md` Milestone 0 carried "Decide and pin: message broker choice (Kafka vs RabbitMQ vs NATS)" as an open bootstrap task. This is not a cosmetic choice — ADR-0006's dual-version event emission and ADR-0007's per-subscriber `event_versions` map assume specific delivery semantics (durable per-consumer position, replay, dead-lettering), and ARCHITECTURE.md §3's "Pull delivery path (subscriber-initiated fetch, for legacy downstream systems that can't accept pushes)" requires a broker model that supports consumer-initiated, durable, replayable reads — not just fire-and-forget fan-out.

## Decision

Use RabbitMQ as the platform's message broker, for both the internal "committed"/"state changed" event bus (ADR-0006) and the outbound per-subscriber delivery queues (ARCHITECTURE.md §2.6, §3).

## Alternatives considered

- **Kafka** — strongest fit for high-throughput log-style replay and long retention, but heavier operationally (ZooKeeper/KRaft, partition rebalancing, broker sizing) for what this platform actually needs: per-subscriber durable queues with retry/dead-letter semantics, not a shared append-only log consumed by many independent readers at different offsets. Rejected as more operational surface than the delivery model requires.
- **NATS (with JetStream)** — lighter-weight than both, but per-subscriber dead-letter handling, delayed retry/backoff, and the pull-consumer model ARCHITECTURE.md §2.6 needs are less mature and less battle-tested than RabbitMQ's native support for the same. Rejected on operational maturity for this specific delivery pattern, not on throughput.
- **RabbitMQ** — chosen. Native support for per-subscriber queues, dead-letter exchanges, and delayed retry maps directly onto ADR-0007's per-subscription retry_policy and Distribution Service's push/pull split, without needing to build queue-per-subscriber semantics on top of a log-shaped primitive.

## Consequences

- Distribution Service's per-subscriber fan-out (ARCHITECTURE.md §2.6) maps naturally onto one RabbitMQ queue per active subscription, with dead-letter exchanges implementing the retry/dead-letter requirement directly rather than as application-level bookkeeping.
- The internal "committed"/"state changed" event bus (consumed by Search Service and Distribution Service, ADR-0006) also runs on RabbitMQ — a topic exchange keyed on `entity_type` + `event_name` is sufficient for both consumers; there is no need for a second broker technology for the internal bus versus outbound delivery.
- KEDA autoscaling (ARCHITECTURE.md §7) for replication/outbound consumers scales on RabbitMQ queue depth, which KEDA supports natively via its RabbitMQ scaler — no custom scaling metric needed.
- Retention/replay is bounded by what's sitting in a queue, not a durable log — if a future requirement needs long-window replay (e.g. "resend everything from the last 30 days" for a newly onboarded subscriber), that's not a RabbitMQ-native capability and would need to be served from `entities`/`entity_drafts` directly rather than from the broker. Flagged as a known limitation, not solved here.
- ADR-0007's `event_versions` per-subscriber map still requires Distribution Service's dispatcher to transform a committed entity into the right shape per subscriber before publishing — RabbitMQ doesn't change that requirement, it only provides the queue/delivery mechanism underneath it.

## Revisit when

A concrete requirement emerges for long-window event replay (beyond what a live queue holds) or for a shared consumer-group read pattern across many independent internal readers — at that point Kafka's log-retention model becomes relevant again and this decision should be reopened against that specific case, not reopened speculatively.
