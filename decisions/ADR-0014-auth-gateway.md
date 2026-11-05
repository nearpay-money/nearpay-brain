# ADR-0014: Traefik gateway with ForwardAuth to identity; short-lived internal JWTs

- **Status:** Accepted · **Date:** 2026-11-05 · **Invariants:** INV-8

## Context
All public traffic needs uniform authentication (API keys, publishable key + client secret, dashboard sessions), rate limiting and routing to several services, without each service re-implementing key lookup.

## Decision
- Traefik routes by path prefix to services. Every request passes ForwardAuth to `identity /authz`, which validates the credential (hash lookup: local LRU → Valkey → Postgres **read replica**, so a primary failover does not stop authentication), applies rate limits (Valkey token buckets; if Valkey is down, conservative per-pod limits) and returns an internal JWT (Ed25519 key held in memory, rotated daily from OpenBao; 60 s TTL; claims: `account_id`, `livemode`, `principal`, `scopes`, `request_id`). Key revocations propagate via `identity.api_key.revoked` events (cache TTL ≤ 30 s as backstop).
- Services verify the JWT locally (JWKS cached), enforce scopes, and always filter by `account_id` + `livemode`.
- Session-only endpoints (destinations, keys, team) reject API-key principals.
- Dashboard is served same-origin via `dashboard.nearpay.money/api/*` to avoid cross-site cookies.

## Consequences
- ➕ One auth implementation; consistent limits; services stay simple.
- ➖ `identity` is on every request's path → horizontally scaled, cached; `authz` p99 budget 5 ms.

## Alternatives considered
- **Envoy Gateway + ext_authz:** more capable, more complex; Traefik suffices.
- **Auth in each service via shared library:** duplicated caches and rate-limit state.

## Revisit when
Gateway needs (gRPC transcoding, advanced traffic policies) exceed Traefik.
