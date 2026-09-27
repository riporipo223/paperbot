# 23 — Testing Strategy

| Field | Value |
|---|---|
| Status | Draft v0.1 |
| Upstream | [02](02-product-requirements.md), [03](03-system-requirements.md), all domain specs |
| Downstream | Every build phase ("Tests" section), [CONTRIBUTING.md](../CONTRIBUTING.md) |
| Used by phases | All; 23 (consolidation) |

## 1. TDD is mandatory

Every implementation task follows **Red → Green → Refactor**:

1. **Red**: write the smallest test that expresses the next behaviour from the spec (cite the requirement ID in the test name, e.g. `FR-EXE-004`). Run it and watch it fail for the right reason.
2. **Green**: write the minimum code to pass.
3. **Refactor**: improve the design with all tests green. No behaviour change.
4. Commit (tests and code together) when green.

Exceptions (tests after code) are allowed only for: generated code, pure configuration files, and exploratory spikes (spike code is thrown away or re-implemented test-first). Build phase docs list the tests to write *before* each task's implementation.

## 2. Test pyramid and levels

| Level | Scope | Tools | Location | Runs in |
|---|---|---|---|---|
| **Unit** | Pure domain functions (economics, scoring, features, risk checks, journal templates, decoders, state machines) | Vitest, fast-check | `modules/*/domain/__tests__`, `packages/*/src/**/__tests__` | `pnpm test` (every commit) |
| **Contract** | Provider payload schemas vs recorded fixtures; API responses vs `api-contract`; DB enum values vs core enums | Vitest + fixtures | `modules/ingestion/**/__tests__/contract`, `apps/engine/test/contract` | `pnpm test` |
| **Integration** | Module + real Postgres (Testcontainers), repositories, persister, partition manager, ledger triggers | Vitest + Testcontainers | `modules/*/adapters/__tests__`, `apps/engine/test/integration` | `pnpm test:integration` (CI) |
| **Data** | Migrations up on an empty DB; constraints reject invalid rows; partition creation/drop; retention/archive round-trip | Vitest + Testcontainers | `packages/db/test` | CI |
| **Event-stream** | Pipeline behaviour: dedup, ordering, lateness, backpressure, flush barrier, reconnect/gap handling with a fake WS server | Vitest + fake WS server (`ws`) + fake timers/SimulatedClock | `modules/pipeline/__tests__`, `modules/ingestion/**/__tests__` | CI |
| **Simulation (scenario)** | End-to-end headless engine over fixture event streams → expected decisions/fills/PnL | Scenario harness (§4) | `apps/engine/test/scenarios` | CI |
| **Backtesting/replay** | Live-vs-replay equivalence, determinism, as-of-time correctness | Scenario harness | `apps/engine/test/replay` | CI |
| **End-to-end** | Dashboard + engine (fixture replay feeding a "live" run) | Playwright | `apps/dashboard/e2e` | CI (main branch + PRs touching dashboard/api) |
| **Failure recovery** | Chaos scenarios (§5) | Scenario harness + fault injection | `apps/engine/test/chaos` | CI (nightly full set, PR subset) |
| **Performance** | Throughput and latency under load with recorded streams | Custom bench runner | `apps/engine/bench` | Manual/nightly. Results in `reports/perf/`. |

## 3. Deterministic fixtures

- **Recorded provider fixtures** (`fixtures/providers/<provider>/<case>.json`), captured in Phase 03/07 from real mainnet via recorder scripts, then sanitized (no API keys in URLs). Each has a `README.md` with capture date, command and slot range.
- **Venue fixtures** (`fixtures/venues/<venue>/`): account data (base64) + expected decoded reserves; logs + expected trades; real trades with observed outputs for economics golden tests.
- **Scenario streams** (`fixtures/scenarios/<name>.ndjson`): sequences of `EventEnvelope`s (normalized) with `receivedAt` timelines. They are human-authored with a builder DSL (`scenario().wallet('A',0.82).buy(...).at('+1.2s')...`), plus some derived from real recordings.
- Fixtures are immutable. A behaviour change that alters golden outputs requires an explicit golden update with a reviewer note.

## 4. Scenario harness

`createTestEngine({ config, seed, fixtures })` builds the full module graph with:
- `SimulatedClock` + scheduler
- `ReplayEventSource` (from the fixture) or fake live adapters
- a real Postgres (Testcontainers) or an in-memory repository implementation (unit-speed mode; both implement the same ports, and a subset of scenarios runs against both to prevent drift)
- `FakeLlmClient` (scripted outputs/timeouts)
- a network guard that fails the test on any outbound connection

Assertions use query helpers: `expectDecisions(...)`, `expectFills(...)`, `expectPortfolio(...)`, `expectTraceComplete(...)`.

### 4.1 Required scenarios (golden)

| ID | Scenario | Expected |
|---|---|---|
| S01 | Wallet buys → wave confirms → entry → price rises → take profit/scale-out → trailing exit | Exact fills, fees, realized PnL (hand-computed, stored in the fixture's `expected.json`) |
| S02 | Wallet buys → liquidity disappears during latency | Entry order FAILED (INSUFFICIENT_LIQUIDITY or SLIPPAGE_EXCEEDED). No position. Fee-only ledger entry if applicable. |
| S03 | Provider disconnects mid-wave → reconnect → missed swaps backfilled as late | Gap recorded. Wave DEGRADED → no entry during the gap. Late events don't retro-trigger. |
| S04 | Stale pool state at execution time | FAILED(STALE_MARKET_DATA). No fill. |
| S05 | Stop loss hit | Exit at the correct time/price. Loss recorded. Consecutive-loss counter increments. |
| S06 | Max hold time exit | Exit at `opened_at + max_hold` on the simulated clock |
| S07 | Smart wallets sell → smart exit | Exit reason SMART_EXIT |
| S08 | Daily loss limit reached | Subsequent signals REJECT with `ENTRIES_PAUSED:DAILY_LOSS_LIMIT`. Exits still work. |
| S09 | Duplicate events from two sources | Single swap. Single wave update. |
| S10 | Out-of-order pool states | Older slot ignored. Price correct. |
| S11 | AI VETO mode: veto, timeout (PROCEED), invalid schema | REJECT (AI_VETO), ENTER, ENTER + INVALID_SCHEMA recorded |
| S12 | Over-extended price at confirmation | No signal (G_EXTENSION) |
| S13 | Token with mint authority present | Wave REJECTED / risk REJECT (RSK_MINT_AUTHORITY) |
| S14 | Engine restart with a SUBMITTED order | Recovery per [16](16-paper-trading-spec.md) §9 |
| S15 | Pump curve completes (migration) while holding | Primary pool switch. Exit routed to the new pool. |
| S16 | DB outage during a wave | Entries paused (PERSISTENCE_BACKLOG). No order without write-ahead. |
| S17 | Exit with stale data then recovery | EXIT_BLOCKED_STALE, then fill once FRESH. Never filled while stale. |
| S18 | Unsupported pool type token | REJECTED (UNSUPPORTED_POOL_TYPE) |

## 5. Failure-mode test matrix

Maps the failure list from the product brief to tests:

| Failure | Test(s) |
|---|---|
| API outage | REST poller circuit breaker tests. Scenario with DexScreener 5xx → DEGRADED snapshots. |
| WebSocket disconnect | Fake WS server drop → reconnect/backoff/resubscribe test. S03. |
| Provider rate limit | 429 with/without Retry-After → limiter backoff, status RATE_LIMITED |
| Duplicate event | Dedup unit + S09 |
| Missing event | Gap detection + backfill (S03) |
| Out-of-order event | S10 + lateness unit tests |
| Invalid token | Decoder/validation rejects → dead letter. Token unresolved → no wave. |
| Invalid pool | Pool decode failure → pool unsupported → REJECT |
| Stale price | S04, S17, quality transition unit tests |
| Liquidity collapse | S02, wave hard invalidation test, LIQUIDITY_EXIT test |
| Database outage | S16. Persister buffer tests. |
| AI provider outage | S11 timeout. Circuit breaker unit test. |
| AI timeout | S11 |
| Malformed AI response | S11 invalid schema |
| Clock drift | Clock offset monitor unit test (injected offsets) → pause at the halt threshold |
| Frontend disconnect | Dashboard stream hook reconnect + gap resync (component test). Engine client-queue overflow → snapshot resync. |
| Server restart | S14. Portfolio rebuild test. |
| Partial system failure | Module handler throws → dead letter + circuit → entries paused (unit + scenario) |

## 6. Coverage and quality gates (CI)

- Line coverage: domain folders ≥ 90%, overall engine ≥ 80%, dashboard ≥ 60% (components with logic).
- Mutation testing (Stryker) on `economics`, `portfolio/domain`, `risk/domain`, run nightly with a target mutation score ≥ 70%. Advisory initially, gating from Phase 23.
- No skipped tests on main (`it.skip`/`describe.skip` banned by lint, except with a `TODO(issue#)` tag on a tracked issue).
- Flaky test policy: a flaky test is a bug. Fix the root cause. Never add retries to hide it.

## 7. Performance tests

- **Pipeline throughput**: replay a recorded high-activity hour at MAX speed. Target ≥ 20× real time and ≥ 2,000 events/s sustained through bus + market-state + wave (without DB trading writes).
- **Latency under load**: the live path with a fake WS server emitting 200 events/s for 10 minutes. Stage p95 within the provisional ceilings ([03](03-system-requirements.md) §2). Event loop lag p99 < 50 ms.
- **Persister**: 10,000 events/s burst for 60 s. The buffer drains, with no loss.
- Results recorded in `reports/perf/<date>.md` with host specs.

## 8. What is never done in tests

- No calls to real providers (network guard). Real calls only happen in explicitly-invoked recorder scripts outside the test suite.
- No `sleep`-based waiting. Use the simulated clock / fake timers.
- No shared mutable global state between tests. Each test gets its own schema (Testcontainers) or a transaction rollback.
