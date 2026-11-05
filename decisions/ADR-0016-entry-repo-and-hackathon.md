# ADR-0016: Thin `nearpay` entry repo; NEAR Hackathon compliance

- **Status:** Accepted · **Date:** 2026-11-05 · **Amends:** ADR-0001 (adds one repo; polyrepo stays)

## Context
NearPay is submitted to the NEAR Hackathon (rules: https://nearhackathon.com/HACKATHON.md, read 2026-11-05). The submission takes exactly **one** public repo URL (unique per repo), repos must be public by **2026-11-14 23:59 UTC**, every repo needs a LICENSE, and the README must let a machine reader and a human reproduce the product ("do X → see Y", no marketing). Libraries, templates and AI-generated code must be listed in the README. The owner rejected a monorepo and chose a thin entry repo.

## Decision
1. Add repo **`nearpay`** (the hackathon `projectUrl`): `README.md` (what, problem, how NEAR is used, reproducible steps), `repos.json` (manifest of all repos with roles and pinned refs), `scripts/bootstrap.sh` (clones/updates every sibling repo from `NEARPAY_GIT_BASE`, default `https://github.com/nearpay-money`), `scripts/up.sh` (runs `nearpay-deploy/compose` with the chosen profile), `HACKATHON.md` (tracks, window, disclosures), `LICENSE`, `AGENTS.md`. No source code, no submodules.
2. **All repos:** Apache-2.0 `LICENSE` (done), README "Built with" section listing libraries and stating that code and docs are AI-assisted (Claude), per FAQ Q-005.
3. **Tracks:** Real-World Money (`realworld`) and Agent Economy (`agents`).
4. **Window:** all work started after the 2026-10-19 announcement, so the full history counts (Q-001); no baseline tag is needed.
5. **Human-only actions** (rules + our policy): pushing, making repos public, posting on X, submitting the form, signing transactions, spending tokens, mainnet deployment. Agents prepare; the human executes or explicitly confirms each.
6. The submission JSON payload is never committed (personal data).
7. **Demo:** Stage 1 mainnet demo runs locally (Q-012 allows local-only with a reproducible README and a video in the X post); hosted demo comes with the final netcup stage.

## Consequences
- ➕ Judges and the AI screener get one URL and one command; polyrepo boundaries stay intact.
- ➖ The entry repo must be kept in sync with the repo list → `repos.json` is generated from the same definition as `architecture/repositories.md`.

## Revisit when
The hackathon ends; afterwards `nearpay` remains the public "getting started" repo.
