# BUILD MASTER PLAN

The complete implementation sequence for Paperbot, for an AI coding agent. Each phase links to its detailed document in `/build`. **If this file and a phase document ever disagree, the phase document wins, and this file must be corrected.** This file is derived from the phase documents.

> **Status:** Documentation stage complete. Implementation MUST NOT begin until the operator approves the documentation. See [build/PHASE-STATUS.md](build/PHASE-STATUS.md).

## 1. Execution loop (every phase)

```text
READ DOCUMENTS (CLAUDE.md → DOCUMENTATION-INDEX.md → phase "Documents to read")
        ↓
SELECT CURRENT PHASE (first phase in build/PHASE-STATUS.md not DONE whose dependencies are all DONE)
        ↓
READ PHASE DOCUMENT (build/phase-XX-*.md)
        ↓
INSPECT REPOSITORY (reconcile with the phase "Inputs"; report discrepancies before coding)
        ↓
IMPLEMENT TASKS (in order; each task: failing test → minimal code → refactor → commit)
        ↓
RUN TESTS (pnpm verify; integration tests where Docker is available)
        ↓
VERIFY ACCEPTANCE CRITERIA (evidence → build/verification/phase-XX.md)
        ↓
UPDATE DOCUMENTATION IF REQUIRED (+ ADR in docs/32 for any deviation)
        ↓
MARK PHASE COMPLETE (build/PHASE-STATUS.md: DONE, date, commit SHA)
        ↓
MOVE TO NEXT PHASE
```

## 2. Sequence at a glance

| # | Phase | Depends on | Milestone |
|---|---|---|---|
| 00 | Repository & development environment | — | M0 |
| 01 | Core foundation | 00 | M0 |
| 02 | Database foundation | 01 | M0 |
| 03 | Provider verification & client foundation | 01 | M1 |
| 04 | Event pipeline core | 02, 03 | M1 |
| 05 | Wallet activity ingestion | 04 | M1 |
| 06 | Reference data | 04 | M1 |
| 07 | Pool streams & venue decoders | 05, 06 | M1 |
| 08 | Market state engine | 07 | M1 |
| 09 | Wallet intelligence | 05, 08 | M2 |
| 10 | Wave detection | 08, 09 | M2 |
| 11 | Economics engine | 01, 07 | M3 |
| 12 | Portfolio accounting | 02, 11 | M3 |
| 13 | Risk engine | 08, 12 | M3 |
| 14 | Paper trading engine | 11, 12, 13 | M3 |
| 15 | Decision engine & strategy runtime | 10, 14 | M3 |
| 16 | AI agent layer | 15 | M4 |
| 17 | Replay & backtesting | 15, 16 | M4 |
| 18 | Engine API | 15, 17 | M5 |
| 19 | Dashboard foundation | 18 | M5 |
| 20 | Dashboard core screens | 19 | M5 |
| 21 | Dashboard intelligence screens | 20 | M5 |
| 22 | Observability hardening | 18 | M6 |
| 23 | Integration, failure & performance testing | 21, 22 | M6 |
| 24 | Deployment | 23 | M6 |
| 25 | Paper trading validation | 24 | M7 |
| 26 | Shadow trading preparation | 25 | M8 |

Rationale for the order: [docs/30-development-plan.md](docs/30-development-plan.md) §2 and [docs/31-build-phases.md](docs/31-build-phases.md).

## 3. Global rules (apply to every phase)

1. **Simulation only.** Never add signing, keypairs, seed phrases, transaction sending, deposits, withdrawals, or a working `RealExecutor`. CI enforces this ([docs/21-security-spec.md](docs/21-security-spec.md) §2).
2. **TDD.** Write the failing test first, citing the requirement ID ([docs/23-testing-strategy.md](docs/23-testing-strategy.md) §1).
3. **No ambient time/randomness/float money.** Use `Clock`, `Rng`, bigint/decimal.
4. **Verify providers before implementing provider-specific behaviour** ([docs/27-data-provider-reference.md](docs/27-data-provider-reference.md) §1). Never fabricate API behaviour or fixtures.
5. **Never silently redesign.** Contradiction → stop, document it (ADR / decision point), ask the operator.
6. **Smallest correct change.** Commit per logical unit, with short English commit messages (`type(scope): summary`).
7. **Docs are the source of truth.** Update the docs in the same change when a contract or behaviour changes.

## 4. Global Definition of Done (in addition to each phase's list)

- [ ] `pnpm verify` passes. CI is green.
- [ ] Tests were written first. Coverage floors are met.
- [ ] `pnpm check:safety` passes.
- [ ] Docs/ADRs updated for any deviation.
- [ ] `build/verification/phase-XX.md` written with evidence (commands, outputs, metrics, deviations).
- [ ] `build/PHASE-STATUS.md` updated.

---

## Phase 00: Repository & Development Environment

**Document:** [build/phase-00-repository-environment.md](build/phase-00-repository-environment.md) · **Milestone:** M0 Foundation · **Depends on:** —  
**Requirements:** FR-SAF-003 (scanner skeleton), NFR maintainability ([03](docs/03-system-requirements.md) §8)

**Objective.** Create a monorepo skeleton where every later phase can add code with tests, linting, type checking, formatting, safety scanning and CI already enforced. **No application logic.**

**Documents to read**

- [04-technical-spec.md](docs/04-technical-spec.md) §1–4, §6, §9
- [24-development-environment.md](docs/24-development-environment.md)
- [21-security-spec.md](docs/21-security-spec.md) §2, §4
- [23-testing-strategy.md](docs/23-testing-strategy.md) §1, §6
- [CONTRIBUTING.md](CONTRIBUTING.md), [CLAUDE.md](CLAUDE.md)

**Tasks**

- 00.1 Workspace skeleton
- 00.2 TypeScript strict config
- 00.3 Prettier + EditorConfig
- 00.4 ESLint with project bans
- 00.5 Safety scanner
- 00.6 Dependency-cruiser
- 00.7 Config immutability check
- 00.8 Docker Compose Postgres
- 00.9 CI workflow
- 00.10 Git hooks (optional)
- 00.11 Verification note

<details><summary>Expected files</summary>

```text
package.json                     # root: scripts (lint, format, typecheck, test, test:integration, check:safety, verify), packageManager
pnpm-workspace.yaml
.nvmrc  .editorconfig  .gitignore  .env.example  .prettierrc  .prettierignore
tsconfig.base.json               # strict flags from 04 §4
eslint.config.js                 # typescript-eslint strict-type-checked + custom restrictions (see Task 00.4)
.dependency-cruiser.cjs          # rules from 04 §6 (initial subset)
vitest.workspace.ts
docker-compose.yml
packages/core/{package.json,tsconfig.json,src/index.ts,src/__tests__/smoke.test.ts}
packages/config/{...same...}
packages/db/{...same...}
packages/api-contract/{...same...}
apps/engine/{package.json,tsconfig.json,src/main.ts,src/__tests__/smoke.test.ts}
scripts/check-safety.ts
scripts/__tests__/check-safety.test.ts
scripts/__tests__/fixtures/banned/*.ts.txt     # files containing each banned pattern (as .txt so they don't compile)
scripts/check-config-immutability.ts
.github/workflows/ci.yml
build/verification/.gitkeep
```

</details>

**Tests**

- Unit: smoke tests, ESLint rule tests, safety scanner tests, depcruise rule test, config immutability test.
- Integration/E2E: none.

**Acceptance criteria**

1. `pnpm install --frozen-lockfile && pnpm verify` exits 0 on a clean clone.
2. Adding `Math.random()` to any engine file makes `pnpm lint` fail.
3. Adding `connection.sendTransaction(` anywhere under `apps/` makes `pnpm check:safety` fail.
4. CI runs on push and passes.
5. No application logic exists yet (only skeletons).

**Definition of done**

- [ ] All acceptance criteria met.
- [ ] Global DoD ([31](docs/31-build-phases.md)) satisfied.
- [ ] `build/PHASE-STATUS.md`: Phase 00 = DONE with the commit SHA.

---

## Phase 01: Core Foundation (`@paperbot/core`, `@paperbot/config`, engine bootstrap guards)

**Document:** [build/phase-01-core-foundation.md](build/phase-01-core-foundation.md) · **Milestone:** M0 Foundation · **Depends on:** 00  
**Requirements:** FR-CFG-001..005, FR-SAF-001, FR-SAF-002 (config + bootstrap layers), FR-SAF-004, NFR-TIME-001/002, NFR-DATA-001

**Objective.** Implement the pure primitives that everything else depends on (time, clock, scheduler, RNG, IDs, money, results, error codes, event envelope, data quality, enums), plus the full configuration system and the engine's startup safety guards (mode guard, forbidden-env guard). After this phase, time, money and randomness have exactly one correct way to be used.

**Documents to read**

- [04](docs/04-technical-spec.md) §4–5, §7–8
- [09](docs/09-real-time-data-architecture.md) §2–3, §5, §7
- [26](docs/26-configuration-reference.md) (entire)
- [21](docs/21-security-spec.md) §2–4
- [28](docs/28-decision-logging-spec.md) §6 (reason code registry)
- ADR-0014, ADR-0016

**Tasks**

- 01.1 EpochMicros and clocks
- 01.2 Scheduler
- 01.3 Seeded RNG with fork
- 01.4 UUIDv7 IdGenerator
- 01.5 Money primitives
- 01.6 Result and error codes
- 01.7 Enums
- 01.8 Addresses
- 01.9 DataQuality
- 01.10 EventEnvelope and schema registry
- 01.11 Canonical JSON + hashing
- 01.12 Config schemas
- 01.13 Config loader
- 01.14 Forbidden-env guard
- 01.15 Mode guard
- 01.16 Logger with redaction
- 01.17 Engine main (bootstrap only)

<details><summary>Expected files</summary>

```text
packages/core/src/
  time/{epoch.ts,clock.ts,system-clock.ts,simulated-clock.ts,scheduler.ts,index.ts}
  rng/{rng.ts,xoshiro128.ts,index.ts}
  ids/{uuidv7.ts,index.ts}
  money/{lamports.ts,raw-amount.ts,decimal.ts,index.ts}
  result.ts
  errors/{codes.ts,domain-error.ts,index.ts}
  events/{envelope.ts,event-types.ts,schema-registry.ts,index.ts}
  quality.ts
  enums.ts
  addresses.ts
  canonical-json.ts
  __tests__/...(one test file per module)
packages/config/src/
  schema/{strategy.ts,engine.ts,index.ts}
  loader.ts  hash.ts  validate.ts  index.ts
  __tests__/{schema.test.ts,loader.test.ts,hash.test.ts,validate.test.ts}
config/base.yaml
config/env/development.yaml  config/env/test.yaml  config/env/production.yaml
config/strategies/smart-money-wave/0.1.0.yaml
apps/engine/src/bootstrap/{env-guard.ts,mode-guard.ts,logger.ts,load-config.ts,banner.ts}
apps/engine/src/bootstrap/__tests__/{env-guard.test.ts,mode-guard.test.ts,logger-redaction.test.ts}
```

</details>

**Tests**

Unit and property tests listed per task. No integration tests.

**Acceptance criteria**

1. All primitives above exist with tests. Domain coverage ≥ 90% for `packages/core` and `packages/config`.
2. `pnpm dev:engine` (with test config) prints the startup banner and exits cleanly.
3. `PRIVATE_KEY=abc pnpm dev:engine` exits non-zero with `SAF_FORBIDDEN_ENV` naming `PRIVATE_KEY`.
4. `run.mode: LIVE` in a strategy file is rejected at load.
5. The config hash of `0.1.0.yaml` is stable and recorded in the verification note.

**Definition of done**

- [ ] Acceptance criteria met.
- [ ] Global DoD satisfied.
- [ ] `config/strategies/smart-money-wave/0.1.0.yaml` hash recorded.

---

## Phase 02: Database Foundation

**Document:** [build/phase-02-database-foundation.md](build/phase-02-database-foundation.md) · **Milestone:** M0 Foundation · **Depends on:** 01  
**Requirements:** FR-SAF-002 (DB layer), FR-CFG-002/003 (tables), NFR-DATA-002/003

**Objective.** Stand up the PostgreSQL foundation: migration tooling, the first four migrations (schemas, ops, strategy core, event log), the partition manager, Kysely types, a connection factory, and Testcontainers-based integration testing. Register the strategy and config versions in the DB.

**Documents to read**

- [07](docs/07-database-schema.md) §1, §3.2 (`market.events`), §3.4 (strategies/versions/config/runs), §3.6, §5–9
- [06](docs/06-data-architecture.md) §3–7
- ADR-0003, ADR-0017, ADR-0020

**Tasks**

- 02.1 Test harness
- 02.2 Migration runner
- 02.3 Migrations 0001–0003
- 02.4 Migration 0004 + partition function
- 02.5 Partition manager
- 02.6 Kysely types + connection
- 02.7 Strategy registry at bootstrap

<details><summary>Expected files</summary>

```text
packages/db/migrations/0001_create_schemas.sql
packages/db/migrations/0002_ops_tables.sql
packages/db/migrations/0003_strategy_core.sql
packages/db/migrations/0004_market_events.sql
packages/db/src/{connection.ts,migrate.ts,partitions.ts,types.generated.ts,index.ts}
packages/db/src/test-utils/{testcontainers.ts,index.ts}
packages/db/test/{migrations.test.ts,constraints.test.ts,partitions.test.ts}
apps/engine/src/bootstrap/strategy-registry.ts
apps/engine/src/bootstrap/__tests__/strategy-registry.int.test.ts
```

</details>

**Tests**

- Data/integration: migrations, constraints, partitions, registry.
- Unit: partition name/range calculation (pure).

**Acceptance criteria**

1. `pnpm db:migrate` on a fresh DB creates all objects in 0001–0004.
2. A LIVE-mode run insert is impossible at the DB level.
3. Partitions are created ahead, and expired ones are dropped (tests).
4. The engine boot registers strategy/config versions idempotently.
5. Integration tests run in CI (Testcontainers).

**Definition of done**

- [ ] Acceptance criteria met. Global DoD satisfied.
- [ ] `types.generated.ts` matches the migrated schema (CI check: regenerate + diff).

---

## Phase 03: Provider Verification & Client Foundation

**Document:** [build/phase-03-provider-verification.md](build/phase-03-provider-verification.md) · **Milestone:** M1 Truthful data · **Depends on:** 01  
**Requirements:** FR-ING-008, FR-SAF-005, [27](docs/27-data-provider-reference.md) §1 policy

**Objective.** Replace provisional provider facts with verified ones, capture real fixtures, and build the generic, safe client foundation: HTTP client with host allowlist, token-bucket rate limiter, retry/circuit breaker, Helius credit budget, the read-only Solana RPC client, and a generic reconnecting WebSocket client. **No normalization or pipeline integration yet.**

**Documents to read**

- [27](docs/27-data-provider-reference.md) (entire; §10 is this phase's checklist)
- [09](docs/09-real-time-data-architecture.md) §11–13
- [10](docs/10-market-data-spec.md) §2, §4
- [21](docs/21-security-spec.md) §2 (RPC allowlist, host allowlist)
- ADR-0009, ADR-0015

**Tasks**

- 03.1 Documentation verification (no code)
- 03.2 Recorder scripts
- 03.3 Record provider fixtures
- 03.4 Record venue fixtures
- 03.5 HTTP client with host allowlist
- 03.6 Token-bucket rate limiter
- 03.7 Retry + circuit breaker
- 03.8 Credit budget
- 03.9 Read-only Solana RPC client
- 03.10 Reconnecting WS client
- 03.11 Contract tests for recorded payloads

<details><summary>Expected files</summary>

```text
apps/engine/src/modules/ingestion/shared/
  http-client.ts          # undici-based; host allowlist; timeouts; redaction
  rate-limiter.ts         # token bucket (clock-driven)
  retry.ts                # retry policy + full jitter backoff
  circuit-breaker.ts
  credit-budget.ts        # Helius credit accounting + pace + conserve-mode signal
  ws-client.ts            # reconnecting WS with ping/pong, backoff, subscription registry
  solana-read-client.ts   # allowlisted JSON-RPC methods only
  __tests__/*.test.ts
scripts/recorders/{helius-logs.ts,helius-account.ts,helius-tx.ts,dexscreener.ts,geckoterminal.ts,coingecko.ts,README.md}
fixtures/providers/**  fixtures/venues/**   (recorded; sanitized)
build/verification/phase-03.md
```

</details>

**Tests**

Unit (clients) + contract (fixtures). No live network in tests (the network guard in the Vitest setup blocks sockets except localhost fake servers).

**Acceptance criteria**

1. [27](docs/27-data-provider-reference.md) has verification status for every fact. Conflicts are resolved or explicitly flagged as blockers.
2. Fixtures exist for all providers and candidate venues, with capture READMEs. There are no secrets in fixtures (gitleaks + a manual grep for the key value).
3. The RPC client cannot send transactions (test). HTTP to non-allowlisted hosts is impossible (test).
4. The rate limiter, retry, circuit breaker, credit budget and WS client are unit-tested, including reconnect.
5. Contract tests parse all recorded fixtures.

**Definition of done**

- [ ] Acceptance criteria met. Global DoD satisfied.
- [ ] DP-04 updated with the verified venue list.
- [ ] Any blocker is reported to the operator with evidence.

---

## Phase 04: Event Pipeline Core

**Document:** [build/phase-04-event-pipeline.md](build/phase-04-event-pipeline.md) · **Milestone:** M1 Truthful data · **Depends on:** 02, 03  
**Requirements:** FR-ING-005, FR-ING-006, FR-ING-009, FR-OBS-002 (latency recorder), NFR-REL-003 (persistence backlog)

**Objective.** Build the shared event pipeline that every source feeds: dedup, sequencing, typed bus with per-subject ordered dispatch and backpressure, async batched persistence with a flush barrier, the `EventSource` abstraction (live and replay stubs), latency recording, dead letters, and the engine module lifecycle (composition root).

**Documents to read**

- [09](docs/09-real-time-data-architecture.md) (entire, especially §1, §3–6, §8–10)
- [05](docs/05-system-architecture.md) §7–9
- [06](docs/06-data-architecture.md) §4
- [07](docs/07-database-schema.md) §3.2 (`market.events`), §3.6 (`ops.*`), §5
- [19](docs/19-backtesting-spec.md) §1, §4 (interface only)
- [22](docs/22-observability-spec.md) §3–5

**Tasks**

- 04.1 Dedup
- 04.2 Sequencer
- 04.3 Lateness / watermark helper
- 04.4 EventBus with per-subject ordering
- 04.5 Event row mapper
- 04.6 EventPersister
- 04.7 EventSource abstraction
- 04.8 Dead-letter sink
- 04.9 LatencyRecorder
- 04.10 SystemEventLog and metrics registry
- 04.11 Composition root and lifecycle
- 04.12 Credit sink
- 04.13 Pipeline throughput smoke benchmark

<details><summary>Expected files</summary>

```text
apps/engine/src/modules/pipeline/
  index.ts ports.ts service.ts
  domain/{dedup.ts,sequencer.ts,subject-queue.ts,lateness.ts,backpressure-policy.ts}
  adapters/{event-persister.ts,event-row-mapper.ts,dead-letter-sink.ts,live-event-source.ts,replay-event-source.ts}
  bus.ts
  __tests__/..., adapters/__tests__/*.int.test.ts
apps/engine/src/modules/observability/
  index.ts latency-recorder.ts system-event-log.ts metrics.ts
  __tests__/...
apps/engine/src/bootstrap/{composition-root.ts,lifecycle.ts,shutdown.ts}
apps/engine/src/modules/ingestion/shared/credit-sink.ts
```

</details>

**Tests**

Unit (domain), integration (persister, replay source, recorder, dead letters), and a benchmark smoke.

**Acceptance criteria**

1. Events from fake live adapters are deduplicated, sequenced, dispatched in per-subject order, and persisted to `market.events` in batches.
2. `flushUpTo` provides the flush-barrier guarantee (test).
3. The persistence backlog policy behaves as specified, including the entry-pause signal hook (`PERSISTENCE_BACKLOG` emitted).
4. `ReplayEventSource` re-emits persisted events identically (envelope equality excluding nothing).
5. Latency samples and system events are persisted.
6. Graceful shutdown drains and flushes.

**Definition of done**

- [ ] Acceptance criteria met. Global DoD satisfied.
- [ ] Benchmark numbers recorded in the verification note.

---

## Phase 05: Wallet Activity Ingestion (Helius)

**Document:** [build/phase-05-wallet-ingestion.md](build/phase-05-wallet-ingestion.md) · **Milestone:** M1 Truthful data · **Depends on:** 04  
**Requirements:** FR-ING-001, FR-ING-005, FR-ING-007, FR-WAL-001 (registry part)

**Objective.** Detect tracked-wallet swaps on Solana in real time and emit normalized `wallet.swap.detected` events. This covers the wallet registry, the subscription manager across limited WS connections, log-based and transaction-based swap decoding (generic balance-delta parser), gap detection and backfill after reconnect, the slot clock, and detection-latency measurement. Execute and record the ADR-0009 measurement.

**Documents to read**

- [10](docs/10-market-data-spec.md) §3, §4.1–4.2
- [09](docs/09-real-time-data-architecture.md) §2.2, §5, §6, §11, §12
- [11](docs/11-wallet-intelligence-spec.md) §2, §7
- [27](docs/27-data-provider-reference.md) §4 (as verified in Phase 03)
- ADR-0009, ADR-0015

**Tasks**

- 05.1 Wallet registry
- 05.2 SubscriptionManager
- 05.3 SlotClock
- 05.4 Log-based swap decoding for wallets
- 05.5 Balance-delta parser
- 05.6 TxEnricher
- 05.7 WalletSwapNormalizer
- 05.8 HeliusWsAdapter + reconnect + gap backfill
- 05.9 Webhook adapter (optional source)
- 05.10 ADR-0009 measurement (manual, real network)
- 05.11 Engine wiring

<details><summary>Expected files</summary>

```text
packages/db/migrations/0005_intel_wallets.sql
apps/engine/src/modules/wallet-intel/{index.ts,ports.ts,service.ts}
apps/engine/src/modules/wallet-intel/adapters/wallet-repository.ts
apps/engine/src/modules/wallet-intel/domain/{wallet.ts,import-parser.ts}
apps/engine/src/modules/ingestion/helius/
  subscription-manager.ts ws-adapter.ts wallet-swap-normalizer.ts tx-enricher.ts
  balance-delta-parser.ts gap-backfiller.ts slot-clock.ts webhook-adapter.ts schemas.ts(extend)
  __tests__/...
apps/engine/src/modules/ingestion/index.ts (public API: watch commands)
scripts/measure-wallet-latency.ts        # ADR-0009 measurement tool (real network; manual)
```

</details>

**Tests**

Unit (parsers, subscription packing, slot clock), contract (recorded fixtures), event-stream (fake WS server reconnect/gap), integration (persistence of swaps + gaps).

**Acceptance criteria**

1. With a real key, the engine detects swaps of tracked wallets and persists `wallet.swap.detected` events with correct amounts (spot-check ≥ 10 against a block explorer; record the links).
2. Reconnect + gap backfill works (fake server test, plus a manual disconnect test on the real network).
3. Detection latency samples are persisted and summarized (p50/p95) in the verification note.
4. ADR-0009 is decided with evidence.
5. No swap event is ever emitted with fabricated amounts (tests for the unparsed path).

**Definition of done**

- [ ] Acceptance criteria met. Global DoD satisfied.
- [ ] ADR-0009 status is `Accepted`.

---

## Phase 06: Reference Data: Venues, Tokens, Pools

**Document:** [build/phase-06-reference-data.md](build/phase-06-reference-data.md) · **Milestone:** M1 Truthful data · **Depends on:** 04 (and 03 fixtures)  
**Requirements:** FR-REF-001, FR-REF-002, FR-REF-003

**Objective.** Resolve and persist token metadata and safety facts, discover a token's pools from DexScreener/GeckoTerminal and on-chain data, maintain the venue registry (with verified fee models and a `supported` flag), and select a primary tradable pool per token, or mark the token unsupported.

**Documents to read**

- [10](docs/10-market-data-spec.md) §4, §7, §10
- [07](docs/07-database-schema.md) §3.1
- [17](docs/17-economics-engine-spec.md) §3 (fee model schema)
- [27](docs/27-data-provider-reference.md) §5–6 (verified)
- ADR-0010

**Tasks**

- 06.1 Venue registry + fee model validation
- 06.2 TokenResolver
- 06.3 SafetyObserver
- 06.4 Pool discovery
- 06.5 PrimaryPoolSelector
- 06.6 Resolution orchestration

<details><summary>Expected files</summary>

```text
packages/db/migrations/0006_ref.sql
apps/engine/src/modules/reference/
  index.ts ports.ts service.ts
  domain/{primary-pool-selector.ts,token-safety.ts,venue.ts,fee-model.ts}
  adapters/{token-repository.ts,pool-repository.ts,venue-repository.ts,safety-repository.ts,
            token-resolver.ts,pool-discovery-dexscreener.ts,pool-discovery-geckoterminal.ts}
  __tests__/...
apps/engine/src/modules/ingestion/dexscreener/{client.ts,schemas.ts(extend)}
apps/engine/src/modules/ingestion/geckoterminal/{client.ts,schemas.ts(extend)}
```

</details>

**Tests**

Unit (selector, safety, fee model), contract (provider payload mapping), integration (repositories, resolution flow against recorded payloads via a mock agent).

**Acceptance criteria**

1. For real tokens bought by tracked wallets, the engine persists token, safety and pools, and selects a primary pool (spot-check ≥ 10 tokens; record results).
2. Unsupported pool types are rejected with a reason, never approximated.
3. Venue `supported` flags reflect Phase 03 verification only.
4. Resolution is single-flight and time-bounded.

**Definition of done**

- [ ] Acceptance criteria met. Global DoD satisfied.

---

## Phase 07: Pool Streams & Venue Decoders

**Document:** [build/phase-07-pool-streams-decoders.md](build/phase-07-pool-streams-decoders.md) · **Milestone:** M1 Truthful data · **Depends on:** 05, 06  
**Requirements:** FR-ING-002, FR-ING-005/006/007 (for pools), FR-MKT-003 (slot monotonic input)

**Objective.** Stream on-chain pool state (reserves) and trades for watched pools, using pure, fixture-verified decoders per supported venue. Emit `pool.state.updated` and `pool.trade.observed` events, handle reconnection with immediate state refresh, and integrate pool watch requests into the subscription manager.

**Documents to read**

- [10](docs/10-market-data-spec.md) §2–5
- [09](docs/09-real-time-data-architecture.md) §6, §8, §11
- [17](docs/17-economics-engine-spec.md) §4 (reserve semantics per venue)
- `fixtures/venues/**` READMEs (Phase 03)
- ADR-0010

**Tasks**

- 07.1 Anchor/borsh helpers
- 07.2 Decoder per verified venue (repeat per venue)
- 07.3 PoolStream
- 07.4 Subscription priorities
- 07.5 GeckoTerminal trades poller
- 07.6 Engine wiring (temporary)

<details><summary>Expected files</summary>

```text
apps/engine/src/modules/market-state/decoders/
  venue-decoder.ts  registry.ts
  pumpfun-curve.ts  pumpswap.ts  raydium-amm-v4.ts  raydium-cpmm.ts   (only verified venues)
  anchor-event.ts   # shared Anchor event discriminator + borsh helpers
  __tests__/<venue>.test.ts
apps/engine/src/modules/ingestion/helius/{pool-stream.ts,pool-state-normalizer.ts,pool-trade-normalizer.ts}
apps/engine/src/modules/ingestion/geckoterminal/trades-poller.ts
apps/engine/src/modules/ingestion/__tests__/pool-stream.test.ts
```

</details>

**Tests**

Unit (decoders, helpers), contract (fixtures), event-stream (fake WS: subscribe/reconnect/refresh), integration (events persisted).

**Acceptance criteria**

1. Each verified venue decoder passes all fixture tests exactly.
2. Watching a real active pool yields a continuous stream of state and trade events. Manual spot-check: 10 trades vs a block explorer, and state vs `getAccountInfo` at the same slot.
3. Reconnect refreshes state immediately (test).
4. Unverified venues have no decoder, and tokens on them are rejected by reference (Phase 06 behaviour intact).

**Definition of done**

- [ ] Acceptance criteria met. Global DoD satisfied.
- [ ] Decoder fuzz tests pass.

---

## Phase 08: Market State Engine

**Document:** [build/phase-08-market-state.md](build/phase-08-market-state.md) · **Milestone:** M1 Truthful data · **Depends on:** 07  
**Requirements:** FR-MKT-001..004, FR-ING-003, FR-ING-004

**Objective.** Build the single internal market state: per-pool reserves, prices, liquidity, trade ring buffers, snapshots and bars, all with per-consumer quality evaluation. Also the market snapshot pollers (DexScreener/GeckoTerminal), the SOL/USD reference, persisted projections, and OHLCV derivation. After this phase, milestone M1 is demonstrable.

**Documents to read**

- [10](docs/10-market-data-spec.md) §1, §5, §6, §8, §9
- [09](docs/09-real-time-data-architecture.md) §7, §8
- [06](docs/06-data-architecture.md) §4
- [07](docs/07-database-schema.md) §3.2 (projections)
- [26](docs/26-configuration-reference.md) `freshness.*`, `market_state.*`

**Tasks**

- 08.1 Price and liquidity math (pure)
- 08.2 QualityEvaluator
- 08.3 Pool market state + slot monotonicity
- 08.4 TradeRingBuffer
- 08.5 BarBuilder (1m/5m/1h)
- 08.6 Snapshot pollers
- 08.7 SOL/USD aggregator
- 08.8 MarketStateStore + View
- 08.9 Projection writers
- 08.10 M1 demo wiring

<details><summary>Expected files</summary>

```text
packages/db/migrations/0007_market_projections.sql
apps/engine/src/modules/market-state/
  index.ts ports.ts service.ts view.ts
  domain/{price.ts,quality-evaluator.ts,trade-ring-buffer.ts,bar-builder.ts,pool-market-state.ts,sol-usd.ts}
  adapters/{pool-state-writer.ts,trade-writer.ts,snapshot-writer.ts,bar-writer.ts,reference-price-writer.ts}
  __tests__/...
apps/engine/src/modules/ingestion/dexscreener/snapshot-poller.ts
apps/engine/src/modules/ingestion/geckoterminal/snapshot-fallback.ts
apps/engine/src/modules/ingestion/sol-usd/{dexscreener-sol.ts,geckoterminal-sol.ts,coingecko-sol.ts,sol-usd-aggregator.ts}
```

</details>

**Tests**

Unit/property (math, buffers, bars, quality), contract (snapshot payloads), integration (writers), event-stream (out-of-order states).

**Acceptance criteria**

1. Market state for real pools matches independent sources: mid price within 0.5% of DexScreener `priceNative` at comparable times (sample of 20; differences explained by timing), and reserves equal to `getAccountInfo` at the same slot.
2. Quality flags transition exactly at the configured thresholds (tests).
3. Projections persist at the specified cadence. Bars render plausible candles (manual check vs GeckoTerminal for the same pool).
4. SOL/USD is available with quality, and divergence detection works.
5. M1 demo recorded in the verification note (logs/screens).

**Definition of done**

- [ ] Acceptance criteria met. Global DoD satisfied.
- [ ] Milestone M1 marked in `build/PHASE-STATUS.md`.

---

## Phase 09: Wallet Intelligence

**Document:** [build/phase-09-wallet-intelligence.md](build/phase-09-wallet-intelligence.md) · **Milestone:** M2 Signals · **Depends on:** 05, 08  
**Requirements:** FR-WAL-001..004

**Objective.** Turn wallet activity into deterministic, versioned wallet scores. This covers history backfill (credit-budgeted), persisted swaps, FIFO round-trip reconstruction, the `wallet-score@1.0.0` model with penalties and confidence shrinkage, as-of-time score resolution, and the live `smart.buy.detected` / `smart.sell.detected` path.

**Documents to read**

- [11](docs/11-wallet-intelligence-spec.md) (entire)
- [07](docs/07-database-schema.md) §3.3
- [26](docs/26-configuration-reference.md) `wallet.*`
- [19](docs/19-backtesting-spec.md) §2 (as-of-time)

**Tasks**

- 09.1 Swap persistence
- 09.2 RoundTripBuilder (pure)
- 09.3 Normalizers and scoring (pure)
- 09.4 Penalty flags (pure)
- 09.5 Score repository with as-of-time
- 09.6 Backfill worker
- 09.7 Early-entry metric C5
- 09.8 Score jobs
- 09.9 SmartActivityDetector (live path)
- 09.10 Auto-promotion (flagged) and mining (flagged)

<details><summary>Expected files</summary>

```text
packages/db/migrations/0008_intel.sql
apps/engine/src/modules/wallet-intel/domain/{round-trips.ts,scoring.ts,penalties.ts,normalizers.ts,smart-activity.ts,mining.ts}
apps/engine/src/modules/wallet-intel/adapters/{swap-repository.ts,round-trip-repository.ts,score-repository.ts,backfill-repository.ts,history-source-rpc.ts,sol-usd-history.ts}
apps/engine/src/modules/wallet-intel/{backfill-worker.ts,score-jobs.ts}
apps/engine/src/modules/wallet-intel/__tests__/...
fixtures/wallets/<case>/{swaps.json,expected-round-trips.json,expected-score.json}
```

</details>

**Tests**

Unit/property (round trips, scoring, penalties), integration (repositories, backfill worker with a mock RPC), scenario (live smart-buy detection).

**Acceptance criteria**

1. Imported real wallets are backfilled within the credit caps, and scores are computed with component breakdowns (record ≥ 5 examples in the verification note).
2. Golden fixture scores match exactly.
3. As-of-time lookup never returns future scores (test).
4. Live smart buys emit `smart.buy.detected` with a score snapshot.
5. Credit usage for backfills is recorded per job.

**Definition of done**

- [ ] Acceptance criteria met. Global DoD satisfied.

---

## Phase 10: Wave Detection

**Document:** [build/phase-10-wave-detection.md](build/phase-10-wave-detection.md) · **Milestone:** M2 Signals · **Depends on:** 08, 09  
**Requirements:** FR-WAV-001..004

**Objective.** Implement the wave engine: per-token state machines seeded by smart buys, the `wave-features@1.0.0` computation, gates, `wave-score@1.0.0`, terminal conditions, and persisted, traceable ENTRY signals behind the flush barrier. No trading yet.

**Documents to read**

- [12](docs/12-wave-detection-spec.md) (entire)
- [09](docs/09-real-time-data-architecture.md) §7–9 (quality, lateness, flush barrier)
- [28](docs/28-decision-logging-spec.md) §3 (signal contents)
- [07](docs/07-database-schema.md) §3.4 (waves, transitions, signals)

**Tasks**

- 10.1 Scenario DSL
- 10.2 Windows helper
- 10.3 Features (pure)
- 10.4 Gates (pure)
- 10.5 Score (pure)
- 10.6 State machine (pure)
- 10.7 WaveService
- 10.8 Golden scenarios
- 10.9 Real-data dry run

<details><summary>Expected files</summary>

```text
packages/db/migrations/0009_waves_signals.sql
apps/engine/src/modules/wave/
  index.ts ports.ts service.ts
  domain/{features.ts,gates.ts,score.ts,state-machine.ts,windows.ts,types.ts}
  adapters/{wave-repository.ts,signal-repository.ts}
  __tests__/{features.test.ts,gates.test.ts,score.test.ts,state-machine.test.ts,service.test.ts}
apps/engine/src/modules/strategy/run-context.ts          # temporary dev RunContext (replaced in 15)
apps/engine/test/scenarios/builders/scenario-dsl.ts       # scenario builder (used from here on)
fixtures/scenarios/wave-*.ndjson + expected.json
```

</details>

**Tests**

Unit (pure domain), scenario (service + SimulatedClock), integration (repositories, flush barrier with the real persister).

**Acceptance criteria**

1. All golden wave scenarios pass with exact expected transitions and scores.
2. Signals are persisted with complete feature snapshots, quality, trigger event IDs and a causation watermark. Every trigger event ID exists in `market.events` (flush barrier test).
3. Replay-order independence within lateness tolerance (test).
4. The live dry run produces waves and signals on real data (evidence recorded).

**Definition of done**

- [ ] Acceptance criteria met. Global DoD satisfied.
- [ ] Milestone M2 marked.

---

## Phase 11: Economics Engine

**Document:** [build/phase-11-economics-engine.md](build/phase-11-economics-engine.md) · **Milestone:** M3 Headless paper trading · **Depends on:** 01, 07 (venue fixtures and verified fee models via 06)  
**Requirements:** FR-ECO-001, FR-EXE-005/006 (model side)

**Objective.** Implement the pure economics module: swap quotes per venue fee model, max input for an impact limit, network/priority/tip costs, ATA rent handling, the latency sampler, the failure model, the MEV model, and `resolveExecution`, which produces a complete, reproducible fill or failure breakdown. No I/O.

**Documents to read**

- [17](docs/17-economics-engine-spec.md) (entire)
- [10](docs/10-market-data-spec.md) §5
- [16](docs/16-paper-trading-spec.md) §5–7 (how the executor uses it)
- `fixtures/venues/*/README.md` (real trades with observed outputs)

**Tasks**

- 11.1 CP AMM math with input fee
- 11.2 Virtual curve math
- 11.3 Golden venue tests
- 11.4 Impact and max input
- 11.5 Network cost
- 11.6 Latency sampler
- 11.7 Failure model ordering
- 11.8 MEV model
- 11.9 resolveExecution
- 11.10 Cost ratio estimate
- 11.11 Property tests

<details><summary>Expected files</summary>

```text
apps/engine/src/modules/economics/
  index.ts
  domain/{cp-amm.ts,virtual-curve.ts,fees.ts,impact.ts,network-cost.ts,latency.ts,failure.ts,mev.ts,resolve-execution.ts,cost-ratio.ts,types.ts,inverse-normal.ts}
  __tests__/{cp-amm.test.ts,virtual-curve.test.ts,golden-venues.test.ts,impact.test.ts,latency.test.ts,failure.test.ts,mev.test.ts,resolve-execution.test.ts,properties.test.ts}
```

</details>

**Tests**

Unit, property, and golden. No integration tests.

**Acceptance criteria**

1. Golden venue tests pass for every venue marked supported. Unsupported venues are documented.
2. All property tests pass (≥ 1,000 runs each).
3. The module has zero imports outside `@paperbot/core` (depcruise).
4. Coverage ≥ 95% lines, mutation score ≥ 70% (Stryker, advisory report).

**Definition of done**

- [ ] Acceptance criteria met. Global DoD satisfied.
- [ ] DP-04 reflects the golden test results.

---

## Phase 12: Portfolio Accounting

**Document:** [build/phase-12-portfolio-accounting.md](build/phase-12-portfolio-accounting.md) · **Milestone:** M3 Headless paper trading · **Depends on:** 02, 11  
**Requirements:** FR-PFL-001..004

**Objective.** Implement the double-entry ledger, positions with average-cost PnL, reservations, marks, portfolio snapshots, invariant checks with fail-safe halting, restart rebuild, and the performance analytics calculator.

**Documents to read**

- [18](docs/18-portfolio-accounting-spec.md) (entire)
- [07](docs/07-database-schema.md) §3.5 (ledger, positions, snapshots)
- [17](docs/17-economics-engine-spec.md) §10 (ExecutionResult shape)
- ADR-0006, ADR-0013, ADR-0014

**Tasks**

- 12.1 Chart of accounts + journal templates (pure)
- 12.2 PositionBook (pure)
- 12.3 Reservations
- 12.4 LedgerService (transactional)
- 12.5 applyExecution
- 12.6 Marks and snapshots
- 12.7 PortfolioSummary read model
- 12.8 computePerformance (pure)
- 12.9 Rebuild from ledger

<details><summary>Expected files</summary>

```text
packages/db/migrations/0010_sim_ledger_positions.sql
apps/engine/src/modules/portfolio/
  index.ts ports.ts service.ts
  domain/{journal.ts,accounts.ts,position-book.ts,reservations.ts,invariants.ts,marks.ts,performance.ts,types.ts}
  adapters/{ledger-repository.ts,position-repository.ts,snapshot-repository.ts}
  __tests__/{journal.test.ts,position-book.test.ts,invariants.test.ts,performance.test.ts,service.int.test.ts,rebuild.int.test.ts}
```

</details>

**Tests**

Unit/property (journal, position book, performance), integration (ledger service, trigger, rebuild).

**Acceptance criteria**

1. Ledger balance is enforced by the app and the DB (tests).
2. The invariant violation halts the run (test).
3. Golden round trip: fund $20 → buy → partial sell → full sell gives exact expected balances and PnL.
4. Rebuild equality.
5. Analytics match hand calculations.

**Definition of done**

- [ ] Acceptance criteria met. Global DoD satisfied.

---

## Phase 13: Risk Engine (+ decision record tables)

**Document:** [build/phase-13-risk-engine.md](build/phase-13-risk-engine.md) · **Milestone:** M3 Headless paper trading · **Depends on:** 08, 12  
**Requirements:** FR-RSK-001..004

**Objective.** Implement the deterministic risk engine: every check in the catalog, entry vs exit semantics, size reduction, and pause-reason signalling, all persisted as auditable risk decisions. Also create the `strategy.decisions` table now, because orders (Phase 14) reference both decisions and risk decisions.

**Documents to read**

- [15](docs/15-risk-engine-spec.md) (entire)
- [17](docs/17-economics-engine-spec.md) §4.3, §8
- [18](docs/18-portfolio-accounting-spec.md) §6–7 (summary inputs)
- [28](docs/28-decision-logging-spec.md) §3, §6

**Tasks**

- 13.1 Check framework
- 13.2 Entry checks (table-driven)
- 13.3 Token safety checks
- 13.4 Exit checks
- 13.5 Reduction algorithm
- 13.6 Pause signalling
- 13.7 RiskService
- 13.8 Property tests

<details><summary>Expected files</summary>

```text
packages/db/migrations/0011_risk_decisions.sql
apps/engine/src/modules/risk/
  index.ts ports.ts service.ts
  domain/{evaluate.ts,checks/entry/*.ts,checks/exit/*.ts,checks/token/*.ts,reduction.ts,types.ts}
  adapters/risk-decision-repository.ts
  __tests__/{checks.table.test.ts,reduction.test.ts,evaluate.test.ts,properties.test.ts,service.int.test.ts}
apps/engine/src/modules/strategy/adapters/decision-repository.ts
```

</details>

**Tests**

Unit/table/property, integration (service persistence).

**Acceptance criteria**

1. All checks are implemented and boundary-tested.
2. Risk decisions persist complete check lists.
3. The pause reasons behave as specified.
4. The decisions table exists and is ready for Phase 14/15.

**Definition of done**

- [ ] Acceptance criteria met. Global DoD satisfied.

---

## Phase 14: Paper Trading Engine

**Document:** [build/phase-14-paper-trading-engine.md](build/phase-14-paper-trading-engine.md) · **Milestone:** M3 Headless paper trading · **Depends on:** 11, 12, 13  
**Requirements:** FR-EXE-001..006, FR-ECO-002, FR-SAF-003 (executor boundary)

**Objective.** Implement `TradingExecutor` with the operational `PaperExecutor` (order lifecycle, write-ahead persistence, latency scheduling, execution against pool state at simulated execution time, fills with full economics, ledger posting, idempotency, restart recovery). Also the `ShadowExecutor` and `RealExecutor` stubs and the `ExecutorFactory` mode enforcement.

**Documents to read**

- [16](docs/16-paper-trading-spec.md) (entire)
- [17](docs/17-economics-engine-spec.md) §5–7, §10
- [18](docs/18-portfolio-accounting-spec.md) §4, §6
- [09](docs/09-real-time-data-architecture.md) §7 (FILL freshness)
- [21](docs/21-security-spec.md) §2–3

**Tasks**

- 14.1 Order state machine
- 14.2 ExecutorFactory and stubs
- 14.3 submit(): idempotency, reservation, write-ahead, scheduling
- 14.4 execute(): fill against state at execution time
- 14.5 ATA handling
- 14.6 Recovery
- 14.7 Safety scan and boundary rules

<details><summary>Expected files</summary>

```text
packages/db/migrations/0012_sim_orders_fills.sql
apps/engine/src/modules/execution/
  index.ts ports.ts
  executor.ts                 # TradingExecutor interface
  paper-executor.ts shadow-executor.ts real-executor.ts executor-factory.ts
  domain/{order-state-machine.ts,idempotency.ts,types.ts}
  recovery.ts
  adapters/{order-repository.ts,fill-repository.ts}
  __tests__/{order-state-machine.test.ts,paper-executor.test.ts,executor-factory.test.ts,recovery.int.test.ts,paper-executor.int.test.ts}
```

</details>

**Tests**

Unit (state machine, factory), component (executor with fakes), integration (DB transactionality, trigger, recovery).

**Acceptance criteria**

1. Fills are computed at the simulated execution time from FRESH state only (tests S02, S04-like at the unit level).
2. Every fill reconstructs signal→decision→execution prices and costs.
3. Idempotency and recovery are proven.
4. LIVE/SHADOW are impossible through the factory.
5. The executor has no network access (static rules + tests).

**Definition of done**

- [ ] Acceptance criteria met. Global DoD satisfied.

---

## Phase 15: Decision Engine & Strategy Runtime

**Document:** [build/phase-15-decision-engine.md](build/phase-15-decision-engine.md) · **Milestone:** M3 Headless paper trading · **Depends on:** 10, 14  
**Requirements:** FR-DEC-001..003, FR-RSK-004, FR-LOG-001 (records), FR-LOG-002, FR-CFG-003

**Objective.** Wire the full SIMULATION loop: RunManager (replacing the temporary RunContext), WatchSetPolicy, DecisionEngine (`decision@1.0.0`), OrderPlanner, ExitManager (`exit@1.0.0`), the entry-pause aggregator, deterministic explanation templates, and an AI port with a no-op implementation. The golden end-to-end scenarios S01–S18 (those not requiring the API/dashboard) pass headless. This reaches milestone M3.

**Documents to read**

- [14](docs/14-strategy-engine-spec.md) (entire)
- [28](docs/28-decision-logging-spec.md) (entire)
- [23](docs/23-testing-strategy.md) §4 (scenario harness and the golden list)
- [10](docs/10-market-data-spec.md) §3 (watch-set policy)
- [13](docs/13-ai-agent-architecture.md) §3, §6 (AI port contract only)

**Tasks**

- 15.1 Reason codes + explanation templates
- 15.2 DecisionEngine (pure)
- 15.3 OrderPlanner
- 15.4 EntryPause aggregator
- 15.5 ExitManager
- 15.6 WatchSetPolicy
- 15.7 RunManager
- 15.8 Strategy service wiring (entry flow)
- 15.9 Scenario harness
- 15.10 Golden scenarios S01–S18
- 15.11 Trace builder and audit CLI
- 15.12 Headless live run

<details><summary>Expected files</summary>

```text
apps/engine/src/modules/strategy/
  index.ts ports.ts service.ts
  run-manager.ts watch-set-policy.ts order-planner.ts entry-pause.ts
  domain/{decision-engine.ts,exit-rules.ts,exit-plan.ts,explanations.ts,reason-codes.ts,types.ts}
  ai-port.ts noop-ai-advisor.ts
  __tests__/{decision-engine.table.test.ts,exit-rules.test.ts,explanations.test.ts,entry-pause.test.ts,run-manager.int.test.ts}
apps/engine/test/harness/{create-test-engine.ts,assertions.ts,network-guard.ts}
apps/engine/test/scenarios/{s01-take-profit.test.ts, s02-liquidity-vanishes.test.ts, ..., s18-unsupported-pool.test.ts}
fixtures/scenarios/s01..s18/{events.ndjson,expected.json,README.md}
apps/engine/src/cli/{run-start.ts,run-stop.ts,trace-audit.ts}
apps/engine/src/modules/strategy/trace/{trace-builder.ts,narrative.ts,__tests__/trace-builder.int.test.ts,__tests__/narrative.test.ts}
```

</details>

**Tests**

Unit/table (decision, exits, explanations, pause), integration (run manager, recovery), scenario (golden set).

**Acceptance criteria**

1. Golden scenarios S01–S18 pass (in-memory mode; the S01, S02, S04, S14 and S16 subset also in Postgres mode).
2. Every decision has a complete trace (`pnpm trace:audit` on scenario runs → 1.0).
3. The live headless run completes 24 h without crash. All entries/exits are paper-only, with the fill/latency/cost fields populated.
4. The temporary RunContext is removed. Wave no longer calls ingestion directly.

**Definition of done**

- [ ] Acceptance criteria met. Global DoD satisfied.
- [ ] Milestone M3 marked.

---

## Phase 16: AI Agent Layer

**Document:** [build/phase-16-ai-agents.md](build/phase-16-ai-agents.md) · **Milestone:** M4 Reproducible · **Depends on:** 15  
**Requirements:** FR-AI-001..004, FR-DEC-002, FR-WAL-005

**Objective.** Implement the advisory AI layer behind the `AiAdvisor` port: the LLM client abstraction (Anthropic, Fake, Recorded), prompt registry, zod output schemas, the six agents, budget and circuit breaker, untrusted-input sanitization, and persistence of every call in `strategy.ai_outputs`. Integrate `ENTRY_REVIEWER` into the decision path for VETO mode, with a timeout and deterministic fallback.

**Documents to read**

- [13](docs/13-ai-agent-architecture.md) (entire)
- [21](docs/21-security-spec.md) §7
- [14](docs/14-strategy-engine-spec.md) §3.2 (rules 6–7)
- [26](docs/26-configuration-reference.md) `ai.*`
- The official Anthropic API documentation (verify the SDK usage, tool-use structured output, model IDs, and pricing at implementation time)

**Tasks**

- 16.1 LlmClient port + FakeLlmClient
- 16.2 Output schemas
- 16.3 Sanitization and fencing
- 16.4 Input hashing
- 16.5 Budget + circuit breaker
- 16.6 AnthropicClient
- 16.7 Agents
- 16.8 ENTRY_REVIEWER in the decision path
- 16.9 Async agents wiring
- 16.10 RecordedLlmClient (for replay)
- 16.11 Research CLI
- 16.12 Live smoke (manual)

<details><summary>Expected files</summary>

```text
packages/db/migrations/0013_ai_outputs.sql
apps/engine/src/modules/ai/
  index.ts ports.ts service.ts
  llm-client.ts anthropic-client.ts fake-llm-client.ts recorded-llm-client.ts
  prompt-registry.ts prompts/{entry-reviewer@1.0.0.ts,token-context@1.0.0.ts,wallet-classifier@1.0.0.ts,signal-explainer@1.0.0.ts,post-trade-analyst@1.0.0.ts,research@1.0.0.ts}
  schemas/{entry-reviewer.ts,token-context.ts,wallet-classifier.ts,signal-explainer.ts,post-trade-analyst.ts,research.ts}
  budget.ts sanitize.ts input-hash.ts
  agents/{entry-reviewer.ts,token-context.ts,wallet-classifier.ts,signal-explainer.ts,post-trade-analyst.ts,research.ts}
  adapters/ai-output-repository.ts
  __tests__/...
apps/engine/src/cli/ai-research.ts
```

</details>

**Tests**

Unit (schemas, sanitize, budget, hashing, client with mocks), scenario (decision path), integration (repository).

**Acceptance criteria**

1. All agents record outputs with a status, model, prompt version, hash, latency and cost.
2. VETO semantics and fallbacks are proven by scenarios. ADVISORY has zero effect on decisions (S11 variants).
3. The prompt-injection fixture has no effect on decisions (structural test).
4. The budget cap is enforced.
5. No secrets in prompts (test).

**Definition of done**

- [ ] Acceptance criteria met. Global DoD satisfied.

---

## Phase 17: Replay & Backtesting

**Document:** [build/phase-17-replay-backtesting.md](build/phase-17-replay-backtesting.md) · **Milestone:** M4 Reproducible · **Depends on:** 15, 16  
**Requirements:** FR-RPL-001..004

**Objective.** Run stored events through the unchanged pipeline with a simulated clock and a recorded AI client. Prove live-vs-replay equivalence and determinism, support both orderings, read cold archives, and provide parameter studies with the required sensitivity set and run comparison.

**Documents to read**

- [19](docs/19-backtesting-spec.md) (entire)
- [09](docs/09-real-time-data-architecture.md) §2.4
- [13](docs/13-ai-agent-architecture.md) §10
- [11](docs/11-wallet-intelligence-spec.md) §8

**Tasks**

- 17.1 ReplayDriver
- 17.2 Replay run factory + network guard
- 17.3 Live-vs-replay equivalence
- 17.4 DeterminismVerifier
- 17.5 ArchiveReader
- 17.6 StudyRunner + sensitivity set
- 17.7 Run comparison service
- 17.8 Replay performance

<details><summary>Expected files</summary>

```text
apps/engine/src/modules/replay/
  index.ts replay-driver.ts replay-run-factory.ts determinism-verifier.ts archive-reader.ts study-runner.ts report-writer.ts
  __tests__/...
apps/engine/src/modules/portfolio/compare.ts        # run comparison query service
apps/engine/src/cli/{replay.ts,replay-study.ts,replay-verify.ts}
apps/engine/test/replay/{live-vs-replay.test.ts,timers.test.ts,as-of-scores.test.ts,archive.test.ts,network-guard.test.ts}
```

</details>

**Tests**

Replay tests, archive tests, determinism tests, and the benchmark.

**Acceptance criteria**

1. Live-vs-replay equivalence test passes in CI.
2. `pnpm replay:verify -- --run <live-run-id>` on the Phase 15 24 h run shows no divergence (or only the documented overhead caveat, flagged).
3. Studies generate reports with held-out metrics.
4. Replay makes zero network calls.

**Definition of done**

- [ ] Acceptance criteria met. Global DoD satisfied.
- [ ] Milestone M4 marked.

---

## Phase 18: Engine API (REST + WebSocket)

**Document:** [build/phase-18-engine-api.md](build/phase-18-engine-api.md) · **Milestone:** M5 Operator UI · **Depends on:** 15, 17  
**Requirements:** FR-UI-002 (server side), FR-LOG-001 (trace endpoint), FR-RSK-004 (pause endpoints), NFR-SEC-003

**Objective.** Expose every read model and the few control commands through a Fastify REST API and a WebSocket stream. Both are defined by shared zod contracts in `@paperbot/api-contract`, with authentication, CORS, rate limiting, idempotency, and contract tests. Optionally mount the Helius webhook route.

**Documents to read**

- [08](docs/08-api-spec.md) (entire)
- [21](docs/21-security-spec.md) §5
- [28](docs/28-decision-logging-spec.md) §4
- [22](docs/22-observability-spec.md) §5–6

**Tasks**

- 18.1 Contracts package
- 18.2 Server skeleton + error handling + health/ready/metrics
- 18.3 Auth
- 18.4 CORS, rate limit, helmet, API version
- 18.5 Read routes (grouped by domain; one commit per group)
- 18.6 Decision trace route
- 18.7 Command routes
- 18.8 WS hub
- 18.9 Webhook route (conditional)
- 18.10 OpenAPI/JSON Schema export (optional but recommended)

<details><summary>Expected files</summary>

```text
packages/api-contract/src/{common.ts,system.ts,runs.ts,config.ts,portfolio.ts,orders.ts,strategy.ts,market.ts,wallets.ts,analytics.ts,auth.ts,stream.ts,index.ts}
apps/engine/src/modules/api/
  index.ts server.ts
  plugins/{auth-service-token.ts,auth-ws-token.ts,cors.ts,rate-limit.ts,helmet.ts,idempotency.ts,error-handler.ts,api-version.ts}
  routes/{system.ts,runs.ts,control.ts,config.ts,portfolio.ts,positions.ts,orders.ts,waves.ts,signals.ts,decisions.ts,risk.ts,ai.ts,market.ts,wallets.ts,analytics.ts,auth.ts,webhooks.ts,health.ts,metrics.ts}
  ws/{hub.ts,channels.ts,client-session.ts,coalescer.ts}
  __tests__/{contract/*.test.ts,auth.test.ts,ws-hub.test.ts,idempotency.test.ts,cors.test.ts}
```

</details>

**Tests**

Contract, integration (routes over a seeded DB), and WS protocol tests.

**Acceptance criteria**

1. Every endpoint in [08](docs/08-api-spec.md) is implemented and contract-tested. Deviations are documented.
2. The WS protocol behaves per [08](docs/08-api-spec.md) §5 (tests).
3. The security controls are tested (auth, CORS, rate limit, idempotency, no stack traces).
4. There is no endpoint capable of real execution or signing (review + safety scan).

**Definition of done**

- [ ] Acceptance criteria met. Global DoD satisfied.

---

## Phase 19: Dashboard Foundation

**Document:** [build/phase-19-dashboard-foundation.md](build/phase-19-dashboard-foundation.md) · **Milestone:** M5 Operator UI · **Depends on:** 18  
**Requirements:** FR-UI-004, FR-SAF-006, FR-UI-002 (client side), FR-UI-003 (global banner)

**Objective.** Create the Next.js dashboard app: authentication, layout and navigation, the design system (tokens + base components), the typed server-side API client, WS token minting, the `useStream` hook with gap detection and resync, global status elements (simulation badge, entries paused chip, degraded banner), and E2E test infrastructure.

**Documents to read**

- [20](docs/20-dashboard-spec.md) §1, §2, §4, §5
- [08](docs/08-api-spec.md) §2, §5
- [21](docs/21-security-spec.md) §6
- ADR-0011, ADR-0012

**Tasks**

- 19.1 App scaffold + lint/depcruise + CSP headers
- 19.2 Formatting utilities
- 19.3 Auth
- 19.4 Engine client (server-side)
- 19.5 WS token route + stream client
- 19.6 Design system components
- 19.7 Shell + degraded banner + entries chip
- 19.8 E2E harness

<details><summary>Expected files</summary>

```text
apps/dashboard/{package.json,next.config.ts,tsconfig.json,tailwind.config.ts,postcss.config.js,middleware.ts}
apps/dashboard/app/{layout.tsx,page.tsx,login/page.tsx,market/page.tsx,wallets/page.tsx,positions/page.tsx,ai/page.tsx,performance/page.tsx,system/page.tsx,runs/page.tsx,decisions/[id]/page.tsx}
apps/dashboard/app/api/{ws-token/route.ts,auth/login/route.ts,auth/logout/route.ts}
apps/dashboard/lib/{engine.ts,session.ts,auth.ts,format/*.ts,stream/{client.ts,use-stream.ts,time-offset.ts}}
apps/dashboard/components/ds/{sim-badge.tsx,quality-dot.tsx,q-value.tsx,kpi-tile.tsx,status-chip.tsx,data-table.tsx,banner.tsx,confirm-dialog.tsx,json-view.tsx}
apps/dashboard/components/shell/{top-bar.tsx,nav.tsx,degraded-banner.tsx}
apps/dashboard/styles/tokens.css
apps/dashboard/__tests__/{format.test.ts,use-stream.test.tsx,q-value.test.tsx,auth.test.ts}
apps/dashboard/e2e/{playwright.config.ts,global-setup.ts,login.spec.ts,shell.spec.ts}
scripts/e2e-engine.ts          # boots engine + DB with fixture replay feeding a LIVE_FEED-like run for E2E
scripts/hash-password.ts
```

</details>

**Tests**

Unit/component (Vitest + Testing Library), E2E (Playwright).

**Acceptance criteria**

1. Login-protected dashboard with the shell, simulation badge, and live system status via WS.
2. Stream gap/resync is proven by tests.
3. No engine token or secret in the client bundle (CI scan).
4. E2E login + shell test passes in CI.

**Definition of done**

- [ ] Acceptance criteria met. Global DoD satisfied.

---

## Phase 20: Dashboard: Portfolio, Positions, Live Market, Wallets

**Document:** [build/phase-20-dashboard-core-screens.md](build/phase-20-dashboard-core-screens.md) · **Milestone:** M5 Operator UI · **Depends on:** 19  
**Requirements:** FR-UI-001 (these screens), FR-UI-002, FR-UI-003, FR-WAL-001 (UI), FR-SAF-006

**Objective.** Implement the four operational screens with live updates: Portfolio, Active Positions (including paper close), Live Market (watchlist, wave candidates, token drawer with chart), and Wallet Intelligence (list, detail, import, status controls).

**Documents to read**

- [20](docs/20-dashboard-spec.md) §3.1–3.4, §5
- [08](docs/08-api-spec.md) §4.3–4.6, §5.2
- [18](docs/18-portfolio-accounting-spec.md) §5.1, §7

**Tasks**

- 20.1 Untrusted string display
- 20.2 Portfolio screen
- 20.3 Positions screen + paper close
- 20.4 Charts
- 20.5 Live Market screen
- 20.6 Wallets screens
- 20.7 E2E

<details><summary>Expected files</summary>

```text
apps/dashboard/app/page.tsx (portfolio), app/positions/page.tsx, app/market/page.tsx, app/wallets/page.tsx, app/wallets/[address]/page.tsx
apps/dashboard/components/portfolio/{kpi-row.tsx,equity-curve.tsx,equity-decomposition.tsx,recent-fills.tsx}
apps/dashboard/components/positions/{positions-table.tsx,position-card.tsx,exit-plan-distances.tsx,close-position-button.tsx}
apps/dashboard/components/market/{watchlist-table.tsx,wave-candidates.tsx,token-drawer.tsx,trades-tape.tsx,safety-facts.tsx}
apps/dashboard/components/charts/{price-chart.tsx,metric-chart.tsx}
apps/dashboard/components/wallets/{wallets-table.tsx,wallet-detail.tsx,score-components.tsx,round-trips-table.tsx,import-dialog.tsx,status-control.tsx}
apps/dashboard/lib/untrusted.ts                    # sanitize/truncate token strings for display
apps/dashboard/__tests__/... ; apps/dashboard/e2e/{portfolio.spec.ts,positions.spec.ts,market.spec.ts,wallets.spec.ts}
```

</details>

**Tests**

Component and E2E.

**Acceptance criteria**

1. All four screens are functional with live updates and quality/staleness indicators.
2. Paper close works end to end, and the trace shows `EXIT_MANUAL`.
3. Untrusted strings are safely displayed (tests).
4. E2E specs pass in CI.

**Definition of done**

- [ ] Acceptance criteria met. Global DoD satisfied.

---

## Phase 21: Dashboard: AI Command Center, Performance, System, Runs & Config, Decision Trace

**Document:** [build/phase-21-dashboard-intel-screens.md](build/phase-21-dashboard-intel-screens.md) · **Milestone:** M5 Operator UI · **Depends on:** 20  
**Requirements:** FR-UI-001 (remaining screens), FR-LOG-001 (UI), FR-RPL-004 (UI), FR-PFL-003 (UI)

**Objective.** Implement the observability and investigation screens: AI Command Center, Performance (with run comparison), System, Runs & Config (start/stop runs, replays, config versions with diff and validation), and the Decision Trace view.

**Documents to read**

- [20](docs/20-dashboard-spec.md) §3.5–3.9
- [28](docs/28-decision-logging-spec.md) §4
- [18](docs/18-portfolio-accounting-spec.md) §7
- [22](docs/22-observability-spec.md) §3–7
- [26](docs/26-configuration-reference.md) (config editor semantics)

**Tasks**

- 21.1 ReasonCodeBadge
- 21.2 Decision Trace view
- 21.3 AI Command Center
- 21.4 Performance
- 21.5 System
- 21.6 Runs & Config
- 21.7 E2E

<details><summary>Expected files</summary>

```text
apps/dashboard/app/ai/page.tsx, app/performance/page.tsx, app/performance/compare/page.tsx, app/system/page.tsx, app/runs/page.tsx, app/runs/[id]/page.tsx, app/runs/config/[id]/page.tsx, app/decisions/[id]/page.tsx
apps/dashboard/components/ai/{agent-status-cards.tsx,signal-feed.tsx,rejected-list.tsx,risk-decisions-list.tsx}
apps/dashboard/components/performance/{metrics-grid.tsx,pnl-distribution.tsx,pnl-by-exit-reason.tsx,cost-breakdown.tsx,holding-time.tsx,latency-percentiles.tsx,compare-table.tsx}
apps/dashboard/components/system/{providers-panel.tsx,streams-panel.tsx,latency-panel.tsx,gaps-table.tsx,system-events-table.tsx,dead-letters.tsx}
apps/dashboard/components/runs/{runs-table.tsx,start-run-form.tsx,start-replay-form.tsx,config-form.tsx,config-diff.tsx,controls.tsx}
apps/dashboard/components/trace/{trace-timeline.tsx,trace-step.tsx,narrative.tsx,completeness.tsx}
apps/dashboard/components/ds/reason-code-badge.tsx
apps/dashboard/__tests__/... ; apps/dashboard/e2e/{trace.spec.ts,performance.spec.ts,system.spec.ts,runs.spec.ts,ai.spec.ts}
```

</details>

**Tests**

Component and E2E.

**Acceptance criteria**

1. All nine screens from [20](docs/20-dashboard-spec.md) §3 exist (with Phase 20).
2. "Why did we enter TOKEN_X?" is answerable in the UI from a position or decision in ≤ 2 clicks.
3. Config editing never mutates an existing version.
4. E2E specs pass.

**Definition of done**

- [ ] Acceptance criteria met. Global DoD satisfied.
- [ ] Milestone M5 marked.

---

## Phase 22: Observability Hardening

**Document:** [build/phase-22-observability.md](build/phase-22-observability.md) · **Milestone:** M6 Production paper · **Depends on:** 18  
**Requirements:** FR-OBS-001..003, FR-LOG-001 (audit job), NFR-TIME-003

**Objective.** Close the observability gaps: a log catalog audit, full metrics coverage, provider health monitoring, the clock offset monitor, alert rules with optional webhook delivery, event loop lag monitoring, the scheduled trace-completeness audit, retention/archive jobs, and optional client render metrics.

**Documents to read**

- [22](docs/22-observability-spec.md) (entire)
- [09](docs/09-real-time-data-architecture.md) §2.3 (clock), §4 (stages)
- [06](docs/06-data-architecture.md) §3 (retention/archive)
- [28](docs/28-decision-logging-spec.md) §7

**Tasks**

- 22.1 Log event registry + coverage test
- 22.2 ProviderHealthMonitor
- 22.3 ClockOffsetMonitor
- 22.4 EventLoopMonitor
- 22.5 LatencyBudgetWatcher
- 22.6 AlertEngine + webhook
- 22.7 Maintenance jobs
- 22.8 Client render metrics (optional)

<details><summary>Expected files</summary>

```text
apps/engine/src/modules/observability/
  provider-health-monitor.ts clock-offset-monitor.ts alert-engine.ts alert-webhook.ts event-loop-monitor.ts latency-budget-watcher.ts
  maintenance/{partition-job.ts,retention-job.ts,archive-job.ts,trace-audit-job.ts}
  log-events.ts               # registry of log event keys (constants)
  __tests__/...
apps/engine/src/modules/api/routes/client-metrics.ts   (optional)
apps/dashboard/lib/stream/render-sampler.ts             (optional)
```

</details>

**Tests**

Unit and integration as listed.

**Acceptance criteria**

1. The log catalog is fully covered, and free-form event keys are banned.
2. Provider health, clock offset, event loop lag and latency budget monitoring are live.
3. The alerts appear on the System page and, if configured, via the webhook.
4. The maintenance jobs run on schedule (tests + one manual run recorded).

**Definition of done**

- [ ] Acceptance criteria met. Global DoD satisfied.

---

## Phase 23: Integration, Failure & Performance Testing

**Document:** [build/phase-23-integration-testing.md](build/phase-23-integration-testing.md) · **Milestone:** M6 Production paper · **Depends on:** 21, 22  
**Requirements:** NFR-REL-001..005, [23](docs/23-testing-strategy.md) §5 and §7, success metrics in [01](docs/01-product-spec.md) §7 (pre-validation)

**Objective.** Consolidate and complete the test suites: the full failure-mode matrix (chaos), performance/load tests, the replay throughput benchmark, full E2E coverage, mutation testing turned into a gate for core domains, and a readiness review before deployment.

**Documents to read**

- [23](docs/23-testing-strategy.md) (entire)
- [03](docs/03-system-requirements.md) §2, §4, §5
- [09](docs/09-real-time-data-architecture.md) §9–14

**Tasks**

- 23.1 Fault injector
- 23.2 Chaos tests (one per failure row)
- 23.3 Performance benchmarks
- 23.4 Long-run soak (recorded data)
- 23.5 E2E completeness
- 23.6 Mutation testing gate
- 23.7 Readiness review

<details><summary>Expected files</summary>

```text
apps/engine/test/chaos/{api-outage.test.ts,ws-disconnect.test.ts,rate-limit.test.ts,duplicate-events.test.ts,missing-events.test.ts,out-of-order.test.ts,invalid-token.test.ts,invalid-pool.test.ts,stale-price.test.ts,liquidity-collapse.test.ts,db-outage.test.ts,ai-outage.test.ts,ai-timeout.test.ts,ai-malformed.test.ts,clock-drift.test.ts,frontend-disconnect.test.ts,server-restart.test.ts,partial-failure.test.ts}
apps/engine/test/harness/fault-injector.ts
apps/engine/bench/{pipeline-throughput.bench.ts,live-latency.bench.ts,persister-burst.bench.ts,replay-throughput.bench.ts,fake-ws-load-server.ts}
.github/workflows/nightly.yml
stryker.conf.json
```

</details>

**Tests**

Chaos, performance, soak, E2E, mutation.

**Acceptance criteria**

1. Every failure mode in [23](docs/23-testing-strategy.md) §5 has a passing chaos test.
2. The performance targets are met or deviations documented with an ADR and mitigation.
3. The soak run shows no leaks, and invariants hold.
4. The mutation score is ≥ 70% on core domains.
5. The readiness checklist is complete.

**Definition of done**

- [ ] Acceptance criteria met. Global DoD satisfied.

---

## Phase 24: Deployment

**Document:** [build/phase-24-deployment.md](build/phase-24-deployment.md) · **Milestone:** M6 Production paper · **Depends on:** 23  
**Requirements:** NFR-SEC-001..004, NFR-REL-001/002, [25](docs/25-infrastructure-deployment.md)

**Objective.** Deploy the engine (Docker) and database to the chosen host and the dashboard to Vercel, with secrets management, TLS, backups, graceful restarts, and a runbook. The resulting production paper environment runs continuously in SIMULATION mode.

**Documents to read**

- [25](docs/25-infrastructure-deployment.md) (entire)
- [21](docs/21-security-spec.md) §4, §8
- [32](docs/32-architecture-decision-records.md) ADR-0018, DP-02, DP-03, DP-08

**Tasks**

- 24.1 Dockerfile
- 24.2 Migration entrypoint
- 24.3 Production compose + Caddy + backups
- 24.4 Release workflow
- 24.5 Host setup (operator + agent)
- 24.6 Dashboard on Vercel
- 24.7 First production start
- 24.8 Graceful restart drill

<details><summary>Expected files</summary>

```text
apps/engine/Dockerfile
apps/engine/.dockerignore
apps/engine/src/migrate.ts
deploy/docker-compose.prod.yml
deploy/Caddyfile
deploy/backup/{Dockerfile,backup.sh}
deploy/engine.env.example            # names only
.github/workflows/release.yml
docs/runbooks/{deploy.md,rollback.md,restore-backup.md,rotate-secrets.md,incident.md}
```

</details>

**Tests**

Container smoke test, migration entrypoint test, and the manual drills (recorded).

**Acceptance criteria**

1. Engine and dashboard are deployed. TLS is valid. The dashboard requires login.
2. Backups run, and one restore was tested.
3. The restart drill passes.
4. No secrets are in the repo, image, or logs (scan the image with gitleaks/trivy secret scanning, and grep the logs).
5. The runbooks exist and were followed once.

**Definition of done**

- [ ] Acceptance criteria met. Global DoD satisfied.
- [ ] Milestone M6 marked.

---

## Phase 25: Paper Trading Validation

**Document:** [build/phase-25-paper-validation.md](build/phase-25-paper-validation.md) · **Milestone:** M7 Evidence · **Depends on:** 24  
**Requirements:** Success metrics [01](docs/01-product-spec.md) §7, FR-RPL-003/004, [22](docs/22-observability-spec.md) §8

**Objective.** Run the system continuously in SIMULATION on real data, establish measured latency baselines (replacing the provisional ceilings), verify operational metrics (traceability, invariants, determinism, uptime), perform out-of-sample parameter and sensitivity studies via replay, and write an evidence-based strategy verdict report. **No code changes to strategy logic during a measurement window.** Fixes create new strategy versions and restart the window.

**Documents to read**

- [01](docs/01-product-spec.md) §7
- [19](docs/19-backtesting-spec.md) §2, §7
- [18](docs/18-portfolio-accounting-spec.md) §7
- [17](docs/17-economics-engine-spec.md) §9 (limitations)
- [22](docs/22-observability-spec.md) §8
- [11](docs/11-wallet-intelligence-spec.md) §8 (out-of-sample wallets)

**Tasks**

- 25.1 Measurement plan (before starting)
- 25.2 Continuous run + daily checks
- 25.3 Latency baselines (after ≥ 72 h)
- 25.4 Determinism check
- 25.5 Studies (replay)
- 25.6 Verdict report (template)

<details><summary>Expected files</summary>

```text
reports/validation/<date>-baselines.md
reports/validation/<date>-strategy-verdict.md
reports/validation/<date>-sensitivity/*.md       (study outputs)
config/strategies/smart-money-wave/<next>.yaml   (only if recommended; new file)
```

</details>

**Tests**

The verification queries are committed as `scripts/validation-queries.sql` (e.g. fills joined with pool state age > fill threshold must return 0 rows).

**Acceptance criteria**

1. ≥ 7 consecutive days of continuous engine uptime with automatic recovery from provider disconnects (NFR-REL-001). ≥ 14 days recommended.
2. Trace completeness 1.0. Zero ledger invariant violations. Zero fills on non-FRESH data (a query-based check in the report).
3. Baselines recorded and budgets configured.
4. The replay determinism check passes on a production day.
5. The verdict report is complete, including limitations and held-out results.

**Definition of done**

- [ ] Acceptance criteria met. Global DoD satisfied.
- [ ] Operator has reviewed the verdict report. Milestone M7 marked.

---

## Phase 26: Shadow Trading Preparation

**Document:** [build/phase-26-shadow-preparation.md](build/phase-26-shadow-preparation.md) · **Milestone:** M8 Shadow-ready · **Depends on:** 25 (verdict report accepted by the operator)  
**Requirements:** FR-EXE-001 (`ShadowExecutor`), FR-SAF-001..005 (still enforced)

**Objective.** Enable `SHADOW` mode: the paper executor remains the source of fills, and additionally, for each order, a **read-only** real-world aggregator quote is fetched and recorded at decision time and at simulated execution time. The divergence between the paper economics and the real routing is measured. **No transaction is built, signed or sent. No keys.**

**Documents to read**

- [29](docs/29-future-live-trading-architecture.md) §3, §6, §7
- [16](docs/16-paper-trading-spec.md) §2
- [21](docs/21-security-spec.md) §2 (host allowlist), §9
- [27](docs/27-data-provider-reference.md) (add the aggregator quote API facts after verification)

**Tasks**

- 26.1 Verify the quote provider (docs + probe) and record the ADR
- 26.2 Path-level allowlist
- 26.3 Quote client + recorder
- 26.4 ShadowExecutor
- 26.5 Divergence analytics
- 26.6 Config + factory enablement
- 26.7 Dashboard panels
- 26.8 Shadow run (≥ 7 days) + report

<details><summary>Expected files</summary>

```text
packages/db/migrations/0014_shadow_quotes.sql
apps/engine/src/modules/execution/shadow/{quote-client.ts,quote-recorder.ts,divergence.ts}
apps/engine/src/modules/execution/__tests__/{shadow-executor.test.ts,quote-client.test.ts,divergence.test.ts}
apps/dashboard/components/orders/shadow-divergence.tsx
apps/dashboard/components/performance/paper-vs-quoted.tsx
fixtures/providers/<aggregator>/*.json
```

</details>

**Tests**

Unit, contract (fixtures), property (ledger equality), and E2E (panel).

**Acceptance criteria**

1. SHADOW runs record real quotes without affecting the paper fills (property test + production check).
2. Transaction-building/sending is impossible (path allowlist tests, safety scan, no signing libraries).
3. The divergence report is produced.
4. All LIVE blocks remain intact (regression tests).

**Definition of done**

- [ ] Acceptance criteria met. Global DoD satisfied.
- [ ] Milestone M8 marked. LIVE remains unimplemented and blocked.

---
