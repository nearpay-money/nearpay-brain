# ADR-0028: An integrity failure on one payment never stops the platform — scoped quarantine, global stop only on corroborated evidence

- **Status:** Accepted
- **Date:** 2026-11-12
- **Deciders:** Owner (directive 2026-11-12: "The service should never stop"); Principal Engineer (lead agent, implementation)
- **Invariants touched:** none weakened (INV-1 non-custody, INV-2 evidence before success, INV-8 livemode hold; every refusal still means no deposit address is shown)
- **Amends:** L-SN-01 (`lld/audits/sentinel.md`) and `lld/sentinel.md` §5 "Classes and reactions"; `architecture/security.md` §6 "Sentinel refuses to co-sign"; runbook RB-03

## Context
On 2026-11-11 at 18:58 UTC, during the closed beta, the global kill switch engaged with no operator involved (Telegram `KillSwitchEngaged`). Sentinel's co-signer engaged it automatically (actor `sentinel`) because one `security`-class refusal concerned a deposit address rails confirmed NearPay had issued. L-SN-01 had already stopped *spoofed* addresses from engaging the switch, but it still let **a single payment's check stop every merchant** until an operator release.

Several `security` reasons can come from causes that are not tampering:
- **Registry view lag** (R5 "recipient is not an active NearPay destination"): sentinel reads identity's view through a read-only role that the HLD places on a replica (§9). Since ADR-0027 a payout wallet is active the moment it is attested, so a quote can name a wallet the replica does not yet show.
- **Configuration drift**: a missing or changed `NP_TREASURY_ACCOUNT` (R6), or a different referral (R3).
- **Representation**: amounts, deadlines and memos compared as strings (`"…:33Z"` vs `"…:33.000Z"`, `null` vs `""`, an EVM address in another case), and timezone-less strings that `Date.parse` reads in the process's local time.
- **Caller-supplied fields** (R4 origin asset, amount, chain, deadline): checkout sends them, so a payer holding a ticket for their **own** attempt could send different values with one request. Rails confirms the address is genuine, so under L-SN-01 this was a **one-request, payer-triggered global stop** (L-SN-01 residual).

A false positive is very costly (every merchant stops until a two-person release). Refusing the payment is enough to protect money: checkout never shows an address that has no signature.

**Root cause (confirmed 2026-11-12):** the refusal was "R4 deposit chain differs" on a genuine BNB Smart Chain deposit. 1Click had started naming tokens with a new bridge suffix (`nep141:bnb-0x….omdep.near`). Sentinel read the id as a NEAR token. It was a false positive caused by a provider naming change, and it was fixed at the root with a platform-wide asset-id parser (spec 0.5.2 `parseAssetId`; `lld/sentinel.md` §18).

## Decision
1. **No automatic global stop from a single payment's check.** A `security` refusal always refuses that payment (409, as before). On an address rails confirms NearPay issued, it is recorded in `integrity_breaches` with full detail and counted in `integrity_refusals_total{livemode,evidence,rule,address}`.
2. **Evidence decides the reaction.** *Pipeline* evidence is a field rails put into the quote, or the provider signed or echoed, that disagrees with NearPay's configuration, identity's registry or the deposit sentinel asked about: R2, R3, R4 (deposit address, memo, address lifetime, address shape), R5, R6. A payer cannot produce it. *Request* evidence covers only fields the caller supplied: R4 origin asset, amount, chain and deadline, and every `intents_transfer` requirement field. Request evidence is refused, recorded and rate-limited per IP, and it **never** quarantines or counts toward the global stop.
3. **Scoped stop.** A pipeline breach on a genuine **live** address also records an audit finding (`check_kind: attestation`, `evt.sentinel.audit_finding.created`: compliance opens a case and notify pages). It also logs `ALERT quote integrity breach` and **quarantines the account(s) involved**: the execution's account (rails' answer, used only to scope) and the account the quote pays (registry). The quarantine lasts `NP_QUARANTINE_TTL_S` (default 30 min). While it holds, sentinel signs nothing for that account (`503 attestation_unavailable`, cached attestations included). Every other merchant keeps working. Quarantines expire by themselves. Ops list, clear and extend them (`quarantines`, `quarantine-clear`, `quarantine-extend`).
4. **Global stop only on corroborated, systemic evidence.** Sentinel engages the switch (actor `sentinel`) only when at least `NP_INTEGRITY_SYSTEMIC_REFUSALS` (default 3) pipeline breaches on genuine live addresses fall within `NP_INTEGRITY_SYSTEMIC_WINDOW_S` (default 900 s) and span at least `NP_INTEGRITY_SYSTEMIC_ACCOUNTS` (default 2) distinct accounts. Both thresholds are at least 2. The decision is taken under an advisory lock. Rationale: a compromised provider key, referral, treasury or rails produces breaches across merchants, while registry lag, one merchant's configuration or one odd quote does not. Once engaged this way, the switch stays engaged until ops release it (the existing two-person / single-operator release), because a corroborated breach is a security event. The reason text and the alert give the runbook.
5. **Reclassify what is not evidence of tampering.**
   - R5 "recipient found nowhere": sentinel re-reads identity's registry after a short pause. A replica further behind than `NP_REGISTRY_MAX_LAG_S` is `unavailable`. A recipient that is one of the account's own wallets (another asset, type or mode) is `stale`, and so is an account whose payout wallets changed within `NP_REGISTRY_RECENT_CHANGE_S`. Both are refused and retried. Only a recipient NearPay never had, read from a current view, stays `security`.
   - "No treasury configured" (R6, intents) is a configuration error: `unavailable` + ALERT.
   - Provider values are compared by meaning: ISO 8601 instants with an explicit offset at full precision, base-unit integers, memo absence (`null` = `""`), and the chain's normal form for deposit addresses. A provider value sentinel cannot read (timezone-less or invalid deadline, missing address deadline, non-integer `amountIn`) is `unavailable`, not `security`. Equality of meaning is kept exactly. The signature, referral, deposit address, origin asset, amount value and treasury fee recipient checks are unchanged.
6. **No new bus subject.** The breach reuses `evt.sentinel.audit_finding.created` (`check_kind` is an open string), and rails' `DepositLookup` already names the execution and account. Quarantine changes are recorded in `quarantines` + `quarantine_log` in one transaction. They are sentinel-internal and consumed by no other service.

## Consequences
- ➕ One false positive costs one merchant up to 30 minutes, not the whole platform. A payer can no longer stop the platform (or quarantine a merchant) by tampering with their own attestation request.
- ➕ Ops get a precise signal (`IntegrityBreachRefused`, `AccountQuarantined`, `KillSwitchEngaged` with the systemic reason) and the full record (`integrity-breaches`).
- ➖ A real compromise that affects **one** merchant only is contained by refusal and quarantine, not by a global stop. That is acceptable: every refused payment shows no address, and ops can still engage the switch with one action.
- ➖ The quarantine gates sentinel's co-signing only. Flows that do not ask sentinel (agent x402 settles, refund legs) stop only with the global switch.
- ➖ Corroboration needs ≥ 3 refusals across ≥ 2 merchants. On a very quiet platform a systemic compromise may be seen later. Every refusal still pages.
- ➖ Changing `NP_TREASURY_ACCOUNT` while checkouts are open turns their quotes (fee to the old treasury) into R6 breaches across merchants, which can reach the systemic threshold. Change the treasury only during a quiet period, or engage the switch first (runbook).

## Alternatives considered
- **Keep the automatic global stop, but debounce it:** lost — it still lets one merchant's or one payer's signal stop everyone.
- **Never engage automatically (operators only):** lost — a compromised provider key or treasury affects every merchant at once, and the switch also stops flows the co-signer does not see.
- **Quarantine a destination instead of the account:** lost for R5, where there is no destination row. Account scope fits the data model with the least new surface and covers all of a merchant's destinations.
- **Ask rails for the expected request values to classify request mismatches:** lost — rails is never a source of a verdict, and the lookup contract would have to change. Treating caller-supplied fields as request evidence needs no trust in rails.

## Revisit when
A real cross-merchant compromise is missed or seen late, the beta volume makes ≥ 3 refusals / 15 min noisy, or the registry moves to a replica with structural lag (tune `NP_REGISTRY_MAX_LAG_S`).
