# CLAUDE.md — AI Coding Agent Contract

You are implementing **Paperbot**, a Solana memecoin trading **simulator** that uses real market data and virtual money. This file is your operating contract. Read it fully before doing anything.

## 0. Current stage

**Documentation stage complete. Awaiting operator approval.** Do not write application code until [build/PHASE-STATUS.md](build/PHASE-STATUS.md) shows the docs are approved and a phase is selected. When asked to implement, work **one phase at a time**.

## 1. Absolute rules (never violate)

1. **Never introduce real-money execution.** No transaction signing, keypairs, seed phrases/mnemonics, `sendTransaction`/`sendRawTransaction`, wallet adapters, deposits, withdrawals, or a functional `RealExecutor`. Not behind flags, not "for testing".
2. **Never introduce secrets or private keys** into code, config files, the database, logs, fixtures, AI prompts or the frontend.
3. **Never fabricate external API behaviour.** Verify provider facts against official docs and live probes ([docs/27-data-provider-reference.md](docs/27-data-provider-reference.md) §1). If you can't verify, stop and report. Never invent fixtures.
4. **Never silently redesign core architecture.** When you find a genuine contradiction, stop, describe it, propose alternatives, and ask. Record the outcome as an ADR.
5. **Never produce a fill from stale/unknown market data**, and never bypass risk checks.
6. **Never let LLM output size, force or execute a trade.** AI can only veto (VETO mode), and it is advisory otherwise.

## 2. Workflow per phase

1. Read, in order: this file → [DOCUMENTATION-INDEX.md](DOCUMENTATION-INDEX.md) → the phase document `build/phase-XX-*.md` → every document in its "Context" section.
2. Check [build/PHASE-STATUS.md](build/PHASE-STATUS.md): all dependency phases must be `DONE`. Mark this phase `IN PROGRESS`.
3. Inspect the repository: existing files, conventions, and the actual state vs the phase "Inputs". Report mismatches before coding.
4. For each task, in order, apply **TDD**: failing test (cite the requirement/task ID) → minimal implementation → refactor → commit.
5. Follow existing conventions ([docs/04-technical-spec.md](docs/04-technical-spec.md) §3–4, [CONTRIBUTING.md](CONTRIBUTING.md)).
6. Implement the **smallest correct change**. No speculative features. Nothing from later phases.
7. Run `pnpm verify` (plus integration/E2E tests where the phase requires them and the environment allows).
8. Verify every acceptance criterion and write the evidence to `build/verification/phase-XX.md`.
9. If behaviour or contracts changed, update the docs in the same change. Add an ADR for deviations.
10. Mark the phase `DONE` in `build/PHASE-STATUS.md` (date + commit SHA).
11. Report: what was done, the evidence, deviations, open questions.

## 3. Engineering rules

- TypeScript strict. zod at every boundary. `bigint`/`decimal.js` for money. No floats for amounts.
- Time only from the injected `Clock`. Randomness only from the seeded `Rng`. No `Date.now()`/`Math.random()` in domain code.
- Modules communicate through the bus or public `index.ts` APIs only.
- Every decision path persists its records (signal, decision, risk decision, order, fill, ledger) with the IDs linking them.
- Replay must behave identically to live. Anything that differs besides EventSource/Clock/LlmClient is a bug.
- Treat token names/symbols/metadata as untrusted input (UI escaping, fenced in prompts).

## 4. Commits

Short English conventional messages (`feat(scope): …`), one logical unit per commit, tests together with code. Push to the branch you were given. Never push to other branches without permission.

## 5. When to ask the operator

Only for: genuine architectural contradictions, missing credentials or accounts (API keys, hosting), open decision points ([docs/32-architecture-decision-records.md](docs/32-architecture-decision-records.md) register), or anything touching real funds (always "no" in this stage). Otherwise, follow the docs and sensible defaults, and record your assumptions in the verification note.

## 6. Useful entry points

| Need | Document |
|---|---|
| Implementation sequence | [BUILD-MASTER-PLAN.md](BUILD-MASTER-PLAN.md) |
| Architecture summary | [ARCHITECTURE.md](ARCHITECTURE.md) |
| Requirements (IDs) | [docs/02-product-requirements.md](docs/02-product-requirements.md) |
| DB schema | [docs/07-database-schema.md](docs/07-database-schema.md) |
| Events, time, quality | [docs/09-real-time-data-architecture.md](docs/09-real-time-data-architecture.md) |
| Config keys | [docs/26-configuration-reference.md](docs/26-configuration-reference.md) |
| Security rules | [docs/21-security-spec.md](docs/21-security-spec.md) |
| Testing rules | [docs/23-testing-strategy.md](docs/23-testing-strategy.md) |
| Decisions and open points | [docs/32-architecture-decision-records.md](docs/32-architecture-decision-records.md) |
