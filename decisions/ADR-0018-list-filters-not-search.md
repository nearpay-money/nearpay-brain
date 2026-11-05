# ADR-0018: Typed filters on list endpoints instead of search endpoints

- **Status:** Accepted
- **Date:** 2026-11-05
- **Deciders:** Owner; Principal Engineer
- **Resolves:** OQ-T-01 (`roadmap/open-questions.md`)
- **Invariants touched:** none

## Context
`architecture/api-conventions.md` §2 promised Stripe-style search endpoints (`/search?query=`) "for dashboard-grade filtering", and the payments LLD listed `GET /v1/payment_intents/search`, but no query grammar was ever designed. Stage 1 lists already take typed filters (`status`, `created[gte]`, `created[lte]`). Richer dashboard filtering is needed from Stage 2.

## Decision
1. NearPay has **no search endpoints and no query language**. Filtering is done with **typed query parameters on list endpoints**, each declared in the route's `query` schema in `nearpay-spec` and validated strictly (unknown parameters are rejected).
2. Filters use one notation: exact match `field=value`; ranges `field[gte]=`, `field[gt]=`, `field[lte]=`, `field[lt]=`; lists of values `field[]=a&field[]=b`; metadata `metadata[key]=value`.
3. `GET /v1/payment_intents` gains `amount` ranges, `currency`, `asset_id` and `metadata[key]` filters in Stage 2, alongside the existing `status` and `created` ranges. Other lists add filters the same way when a screen or integration needs them; every addition is backward compatible.
4. Results stay ordered by `(created desc, id desc)` with cursor pagination (`starting_after` / `ending_before`).

## Consequences
- ➕ One mechanism: typed, documented in OpenAPI, validated, and covered by the compatibility gate; no parser to build or secure.
- ➕ Each filter maps to an index the owning service chooses deliberately.
- ➖ No free-text or OR queries; a merchant who needs them exports data or uses the dashboard's combined filters.

## Alternatives considered
- **Stripe-style search language** — familiar to Stripe users, but needs a grammar, a parser, documentation and search indexing for a need nobody has yet.
- **Leave it open until Stage 2** — rejected: list filter conventions shape Stage 1 contracts now.

## Revisit when
Merchants ask for free-text search across objects (e.g. by customer email), or list filter combinations stop scaling.
