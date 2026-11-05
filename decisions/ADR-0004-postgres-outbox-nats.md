# ADR-0004: Postgres per service + transactional outbox → NATS JetStream

- **Status:** Accepted · **Date:** 2026-11-05 · **Invariants:** INV-4, INV-9

## Context
Money state needs ACID transactions; several services react to the same facts (webhooks, billing, notify, compliance). Self-hosted on netcup with a small ops footprint.

## Decision
- Each service owns one PostgreSQL database (CloudNativePG cluster shared physically, separated by database + role).
- State change + transition row + outbox row are written in one transaction. A relay publishes outbox rows to NATS JetStream (`evt.<domain>.<aggregate>.<event>`, message ID = event ID). Relayed rows are kept 7 days for replay.
- Consumers use an inbox table for dedupe and ignore stale `aggregate_version`.
- Durable jobs (expiry, scheduling, retries) use Postgres (`SKIP LOCKED`) — no separate job system.

## Consequences
- ➕ No lost or phantom events; NATS is rebuildable from outboxes; few components to run.
- ➖ Outbox relay adds ~100–500 ms latency (within NFR-LAT-02).
- ➖ One physical Postgres cluster is a shared failure domain; separated logically and can be split per service later.

## Alternatives considered
- **Kafka/Redpanda:** heavier to operate on a few root servers; unnecessary at v1 scale.
- **Postgres LISTEN/NOTIFY only:** no durable multi-consumer replay.
- **Event sourcing:** stronger audit but higher complexity; transition tables + observations give the audit we need (INV-9).

## Revisit when
Sustained > 2k events/s, or a service needs its own Postgres cluster for load isolation.
