# ADR-0017: Git workflow — `main` plus feature branches merged by pull request

- **Status:** Accepted; decisions 3–5 amended by ADR-0020 (the lead agent reviews and merges pull requests, including money-path pull requests; merging leaves the human-only list)
- **Date:** 2026-11-05
- **Deciders:** Owner; Principal Engineer
- **Amends:** ADR-0016 decision 5 (pushing feature branches and opening pull requests are delegated to the agent)
- **Invariants touched:** none

## Context
Step 1.1 was built on a local branch and fast-forwarded into `main` once its exit criteria passed. The 27 repositories are about to get GitHub remotes under `nearpay-money`. ADR-0016 made every push a human-only action, which turns each step into many manual pushes and leaves no reviewable record of what changed between steps.

## Decision
1. **`main` is the latest working build** of every repository. Nothing is committed to it directly.
2. **Every development step, feature or document change gets its own branch from `main`**, named `stage-<n>/<step>-<topic>` for plan steps (e.g. `stage-1/1.2-services`) or `<kind>/<topic>` otherwise (`docs/…`, `fix/…`).
3. **A branch joins `main` only through a pull request** that the owner reviews and merges. Money-path paths keep the CODEOWNERS rule (two human approvals once the org has them).
4. **The agent may push feature branches and open pull requests** (owner decision 2026-11-05). It never pushes to `main`, never force-pushes a branch someone else pushed to, never merges, and never changes repository visibility.
5. Still human-only (ADR-0016 decision 5, unchanged otherwise): merging pull requests, the first push of `main` to a new remote, making repositories public, posting, submitting, signing transactions, spending funds, mainnet deployment, providing credentials.
6. Until a repository has a remote, finished branches are merged locally by fast-forward with the owner's go-ahead; the branch stays so it can still be pushed and reviewed.

## Consequences
- ➕ Each step is one reviewable pull request per repository; `main` stays releasable.
- ➕ The owner keeps the final say (merge) without doing the mechanical pushes.
- ➖ Cross-repository steps produce several pull requests that must merge together; the step's tracker entry lists them.

## Alternatives considered
- **Owner pushes every branch** — the ADR-0016 default; rejected by the owner as needless manual work.
- **Agent merges after green CI** — rejected: merging into `main` is the owner's review gate.

## Revisit when
More engineers join (branch protection, required reviews per team) or CI starts deploying from `main`.
