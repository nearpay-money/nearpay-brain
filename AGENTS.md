# AGENTS.md — working on NearPay (all repositories)

You are an AI agent contributing to NearPay. These rules apply in every `nearpay-*` repo. Repo-specific rules are in that repo's `AGENTS.md`.

## 1. Load context in this order

1. `nearpay-brain/product/requirements.md` §1 — the invariants. Memorise INV-1…9.
2. The target repo's `README.md` → the brain sections it links.
3. `architecture/domain-model.md` if you touch any money object; `architecture/api-conventions.md` if you touch any API or event.

## 2. Non-negotiables

- **INV-1 Non-custody:** never add code that lets a NearPay key move payer or merchant funds. NearPay keys may pay gas or relay payloads signed by the owner — nothing else.
- **INV-2 Evidence before success:** never mark a payment/refund succeeded from a provider status alone.
- **INV-5 Integer money:** bigint base units + explicit asset/currency; no floats, no `Number` for amounts.
- **INV-8 Livemode:** every query filters by `account_id` and `livemode`; test objects never reach mainnet providers.
- Contracts come from `@nearpay/spec`. Change the spec first (PR in `nearpay-spec`), then implement.
- One transaction per state change: aggregate row + transition row + outbox row.

## 3. How to change things

| You want to… | Do this |
|---|---|
| Add/modify an endpoint or event | PR to `nearpay-spec` (additive within version) → release → bump in consumers |
| Change a state machine | Update `architecture/domain-model.md` + property tests in the owning service in the same change set; money-path CODEOWNERS review |
| Make or reverse a design decision | New ADR in `decisions/` (copy the template); link it from the HLD |
| Add a dependency on another service | Check HLD §5 allowed edges; if new, ADR + Cilium policy change in `nearpay-deploy` |
| Rely on an external fact | Add it to `reference/landscape.md` with source + date; ⚠ + spike if unverified |
| Add a failure mode | Add a stress-test scenario and, if operational, a runbook row |

## 4. Never

- Import another repo's source, read another service's DB, or share tables.
- Put architecture docs inside code repos (link to the brain instead).
- Hold production credentials, run production migrations, or merge to `nearpay-deploy` production overlays without a human.
- Weaken a check in a money path to make a test pass — fix the code or escalate.
- Invent provider behaviour. If 1Click/NEAR docs are silent, write a spike and code defensively (unknown → `held`).

## 5. Definition of done

Spec updated → code + tests (unit, property for state machines, contract) → brain updated → CI green → human review on money paths.
