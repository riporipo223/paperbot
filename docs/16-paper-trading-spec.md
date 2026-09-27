# 16 — Paper Trading Specification

| Field | Value |
|---|---|
| Status | Draft v0.1 |
| Upstream | [17](17-economics-engine-spec.md), [15](15-risk-engine-spec.md), [10](10-market-data-spec.md), [09](09-real-time-data-architecture.md) §7 |
| Downstream | [18](18-portfolio-accounting-spec.md), [29](29-future-live-trading-architecture.md) |
| Used by phases | 14 (implementation), 26 (shadow prep) |

## 1. Purpose

Provide a simulated exchange that behaves like sending a swap to a Solana DEX: time passes, the market moves, fees are charged, transactions can fail, and the result is computed from real pool state at the moment of execution.

## 2. Executor abstraction

```ts
export interface TradingExecutor {
  readonly kind: 'PAPER' | 'SHADOW' | 'REAL';
  submit(order: ApprovedOrder): Promise<OrderHandle>;     // persists + schedules; returns once SUBMITTED is durable
  cancel(orderId: OrderId): Promise<void>;                // only CREATED/SUBMITTED before exec time
  recover(runId: RunId): Promise<RecoveryReport>;          // restart handling (§9)
}
```

| Implementation | Status | Behaviour |
|---|---|---|
| `PaperExecutor` | **Operational (MVP)** | This document |
| `ShadowExecutor` | Stub until Phase 26 | Phase 26: delegates to `PaperExecutor` and also records a read-only "real-world quote" (e.g. Jupiter quote API) for comparison. It never signs or sends. |
| `RealExecutor` | **Permanent stub in this project stage** | Every method throws `LiveTradingDisabledError('SAF_LIVE_DISABLED')`. Not registered in `ExecutorFactory`. No imports of signing libraries. |

`ExecutorFactory.forMode(mode)`: `SIMULATION → PaperExecutor`. `SHADOW → ShadowExecutor` only if `features.shadow_enabled` (false until Phase 26), else throws. `LIVE → throws SAF_MODE_NOT_ALLOWED`. Unit tests assert all three paths, and a static test asserts `RealExecutor` is not referenced by the factory.

## 3. Order model

See `sim.orders` ([07](07-database-schema.md) §3.5). Key fields: `side`, `intent`, `amount_in_raw`, `quote_mid_price`, `quote_expected_out_raw`, `min_out_raw`, `max_slippage_bps`, `priority_fee_lamports`, `idempotency_key`, `status`, `failure_reason`, `sim_latency_us`, `scheduled_exec_at`.

- BUY: `amount_in_raw` is lamports (SOL in). SELL: token raw units.
- **Partial fills**: a single Solana AMM swap is atomic. It fills completely or fails. The paper executor therefore **never produces partial fills for a single order** ([ADR-0022](32-architecture-decision-records.md#adr-0022-no-partial-fills-for-atomic-amm-swaps)). Partial position exits are modelled as separate orders (scale-out), each atomic. The curve-venue edge case where the program clamps output to the remaining real reserves is handled per the verified program behaviour ([17](17-economics-engine-spec.md) §4.2).

## 4. Order lifecycle

```text
 CREATED ──persist──▶ SUBMITTED ──(scheduled_exec_at reached)──▶ FILLED
    │                    │                                 └──▶ FAILED (STALE_MARKET_DATA | SLIPPAGE_EXCEEDED | INSUFFICIENT_LIQUIDITY |
    │                    │                                               SIMULATED_TX_FAILURE | POOL_UNAVAILABLE)
    │                    ├──(cancel before exec)──▶ CANCELLED
    │                    └──(exec not reached by expiry)──▶ EXPIRED   (engine stopped; see §9)
    └──(insufficient funds at reservation)──▶ FAILED (INSUFFICIENT_FUNDS)
```

Legal transitions are enforced in code and by a DB trigger. Every transition is written to `sim.order_transitions`.

## 5. Execution algorithm

```text
submit(order):
  1. Idempotency: if idempotency_key exists → return the existing handle (no duplicate).
  2. Reserve funds (portfolio.reserve): BUY reserves amount_in + est. fees + priority + rent-if-needed.
                                         SELL reserves the token quantity.
     Fail → FAILED(INSUFFICIENT_FUNDS).
  3. Persist order CREATED → SUBMITTED (one DB transaction), submitted_at = clock.now().
  4. Draw latency L = economics.sampleLatency(model, rng.fork('latency').next()); overhead = measured.
     scheduled_exec_at = decided_at + overhead + L ; persist sim_latency_us + scheduled_exec_at.
  5. scheduler.schedule(scheduled_exec_at, () => execute(order.id)).

execute(orderId):
  6. Load order. Status must be SUBMITTED (else no-op; idempotent).
  7. state = marketState.poolStateAt(pool, now)   // live: current state; replay: latest state with replay ts ≤ now
     quality = marketState.quality(pool, 'FILL')
  8. result = economics.resolveExecution({ order, stateAtExec: state, uFail, uMev, uMevFail, ... })
  9. One DB transaction:
       - order → FILLED or FAILED (+ failure_reason)
       - on FILLED: insert sim.fills (with pool_state_event_id, slot, age, full breakdown)
                    mark the market.pool_states row used_for_fill (insert if throttled out)
       - portfolio.applyExecution(result) → ledger transaction(s), position update, release reservation
  10. Emit order.filled / order.failed; portfolio emits position.* and portfolio.updated.
```

- The pool state used **must be FRESH for `FILL`** ([09](09-real-time-data-architecture.md) §7). If it is not → `FAILED(STALE_MARKET_DATA)`, no fees, reservation released. This is the core "no fake fills" guarantee (FR-EXE-004), and it is covered by dedicated tests.
- Random draws use forked RNG streams keyed by `(runId, orderId)` so results do not depend on the order in which other orders drew numbers: `rng.fork('latency:' + orderId)`, `rng.fork('failure:' + orderId)`, `rng.fork('mev:' + orderId)`.

## 6. Price references stored per fill

| Field | Meaning |
|---|---|
| `mid_price_at_signal` | Signal's mid price (entries) or exit-trigger mid (exits) |
| `mid_price_at_decision` | Mid used for the order quote |
| `mid_price_at_execution` | Mid of the state at execution time |
| `exec_price` | Effective price including fees |
| `price_impact_bps`, `adverse_move_bps`, `slippage_bps` | Per [17](17-economics-engine-spec.md) §4.4 |
| fee components, `rent_delta_lamports`, `sol_usd`, `total_cost_usd` | Per [17](17-economics-engine-spec.md) §2 |

Together these let anyone reconstruct *signal price → decision price → execution price → net result* (FR-ECO-002).

## 7. Token account (ATA) handling

- The paper account tracks which mints have an open (virtual) token account.
- First BUY of a mint in a run with no open ATA → `rent_delta_lamports = +rent` (locked from cash into `RENT_DEPOSIT`).
- A SELL that brings the quantity to zero and `economics.close_ata_on_full_exit = true` → `rent_delta_lamports = −rent` (refund to cash), and the close instruction's compute is included in the same tx (no extra base fee).
- A failed first BUY does not create the ATA (the tx reverted), so no rent is locked.

## 8. Safety invariants (tested)

1. No code path in `execution` imports `@solana/kit` transaction-sending APIs, keypair utilities, or HTTP clients to execution venues (dependency-cruiser + safety scan).
2. `PaperExecutor` has no network access: its only dependencies are the DB, market state, economics, portfolio, clock, rng and scheduler.
3. A fill always references a pool state event with quality FRESH at execution time.
4. The sum of reservations never exceeds available cash (property test with the portfolio).
5. Idempotency: submitting the same `idempotency_key` twice creates one order.

## 9. Restart and recovery

On engine start with a `RUNNING` run:

| Order status found | Action |
|---|---|
| `CREATED` (never submitted) | Mark `FAILED(ENGINE_RESTART)`, release the reservation, no fees. The decision is traceable as "not executed". |
| `SUBMITTED`, `scheduled_exec_at` in the future | Re-schedule at the same time |
| `SUBMITTED`, `scheduled_exec_at` passed by ≤ `execution.recovery_grace_ms` (5,000) and a FRESH pool state exists in the event log at or before `scheduled_exec_at` | Execute against that historical state (deterministic) |
| `SUBMITTED`, passed by more, or no suitable state | Mark `EXPIRED` (fees: base + priority, as if the tx was sent but its outcome is unknown; conservative). Release the reservation. Emit an alert. |

The recovery report is logged as a system event and shown on the System page.

## 10. Tests (minimum)

- Happy path BUY and SELL with an exact expected fill (fixture state + fixed `u` values).
- Stale state at execution → FAILED(STALE_MARKET_DATA), no ledger fee entries.
- Slippage breach → FAILED(SLIPPAGE_EXCEEDED) with a fee-only ledger entry.
- Liquidity disappears between decision and execution (fixture) → INSUFFICIENT_LIQUIDITY or SLIPPAGE_EXCEEDED, never a fill at the old price.
- Latency scheduling on the simulated clock: the fill uses the state at `decided_at + L`, not at the decision.
- Idempotent submit. Idempotent execute (double timer fire).
- Recovery table (each row).
- ExecutorFactory refuses SHADOW (flag off) and LIVE. `RealExecutor` methods throw.
