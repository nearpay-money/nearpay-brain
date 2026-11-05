# ADR-0013: TypeScript for services and frontends; Rust for NEAR contracts

- **Status:** Accepted · **Date:** 2026-11-05

## Context
Owner preference: TypeScript and Rust. One language across services, SDKs and frontends maximises reuse of spec-generated types and agent productivity. NEAR contracts are best supported in Rust (`near-sdk`).

## Decision
- Services: Node.js 22 LTS, TypeScript strict, Fastify, `pg` + Kysely (typed SQL, no heavy ORM), zod validators generated from `nearpay-spec`, OpenTelemetry, Vitest + fast-check (property tests).
- Frontends: Vite + React + TypeScript; docs with a static-site generator consuming the OpenAPI.
- Contracts: Rust `near-sdk`, `near-workspaces` integration tests, `cargo-fuzz`/proptest.
- Package manager pnpm; Node version pinned via `.nvmrc`/`packageManager`.
- Rust is also allowed for a service component only if profiling proves TypeScript insufficient (candidate: NEAR indexer).

## Consequences
- ➕ Shared types end-to-end; one hiring/agent profile.
- ➖ Node's single-threaded CPU limits for crypto-heavy work (signature verification at high agent volume) → worker threads or a Rust native module if needed.

## Revisit when
Agent-payment verification exceeds ~2k signatures/s per pod.
