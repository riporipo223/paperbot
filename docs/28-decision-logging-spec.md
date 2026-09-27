# 28 — Decision Logging and Traceability Specification

| Field | Value |
|---|---|
| Status | Draft v0.1 |
| Upstream | [12](12-wave-detection-spec.md), [14](14-strategy-engine-spec.md), [15](15-risk-engine-spec.md), [16](16-paper-trading-spec.md), [07](07-database-schema.md) |
| Downstream | [08](08-api-spec.md) (trace endpoint), [20](20-dashboard-spec.md) (Decision Trace), [22](22-observability-spec.md) |
| Used by phases | 10, 13, 14, 15, 18, 21 |

## 1. Goal

For any decision, answer from stored data alone:

> Why did the system enter / reject / exit TOKEN_X at time T?

Example answer shape:
```text
Wallet A (score 0.82) bought 1.2 SOL at slot 312,000,101 (detected 1.4 s later)
Wallet B (score 0.77) bought 0.8 SOL at slot 312,000,640
Follow-on unique buyers: 14 in 3m12s; tx acceleration 3.4×; buy ratio 0.71
Price +18% since first smart buy (limit 60%); liquidity $38,200 (min $10,000)
All gates passed; wave score 0.76 ≥ 0.70 (wave-score@1.0.0)
Data: pool state FRESH (age 420 ms), trades HEALTHY, SOL/USD FRESH (age 12 s)
AI: ADVISORY, no effect (TOKEN_CONTEXT: no red flags)
Risk: 22/22 checks passed; size $2.00 (0.01124 SOL); impact 41 bps; cost ratio 0.12
Order submitted 38 ms after signal; simulated latency 1.31 s; filled at +2.1% adverse move, 0 bps MEV
```

## 2. The causal chain (what links to what)

```text
market.events (trigger swaps, trades, states)  ◀── signal.trigger_event_ids
intel.wallet_scores (as of decision)            ◀── signal.features.cluster[].scoreId
strategy.waves ── strategy.wave_transitions     ◀── signal.wave_id
strategy.signals ──▶ strategy.decisions ──▶ strategy.risk_decisions
                         │        └──▶ strategy.ai_outputs (ai_output_id)
                         ▼
                     sim.orders ──▶ sim.order_transitions
                         ▼
                     sim.fills (pool_state_event_id ──▶ market.events)
                         ▼
                     sim.ledger_transactions ──▶ sim.ledger_entries
                         ▼
                     sim.positions ──▶ sim.position_transitions
```

Every hop is by explicit ID. No join by "nearest timestamp".

## 3. What each record must contain

| Record | Must contain (in addition to schema basics) |
|---|---|
| Signal | Feature snapshot with values, score breakdown, gates with values and limits, per-input `{quality, ageMs, source}`, trigger event IDs, wallet score IDs used, model versions, `causation_watermark` |
| Decision | Action, ordered reason codes, template explanation, the full `inputs` digest (signal ID, market quality snapshot, portfolio summary, pause state, AI summary), `ai_effect`, model version, `system_latency_us` |
| Risk decision | Every check `{code, passed, value, limit, severity, reducedTo?}`, requested vs approved size, portfolio state snapshot |
| Order | Quote at decision (mid, expected out, impact), slippage settings, priority fee, latency draw, scheduled time |
| Fill | Three mid prices, exec price, all cost components, pool state event ID/slot/age, RNG draw values, economics model version |
| Wave transition | From/to, reason, score, features at transition |

## 4. Decision trace API contract

`GET /api/v1/decisions/:id/trace` returns:

```ts
interface DecisionTrace {
  decision: DecisionRecord;
  run: { id; mode; source; strategyVersion; configVersionId; configHash; codeRef; rngSeed };
  signal?: SignalRecord & {
    triggers: Array<{ event: EventSummary; wallet?: { address; label; scoreAtTime; scoreId; flags } ; detectLatency: { slots; msEst; msBlockTime? } }>;
  };
  wave?: { wave: WaveRecord; transitions: WaveTransition[] };
  ai?: AiOutputRecord | null;
  risk?: RiskDecisionRecord;
  orders: Array<OrderRecord & { transitions: OrderTransition[]; fill?: FillRecord }>;
  position?: PositionRecord & { transitions: PositionTransition[] };
  latency: Array<{ stage: string; durationUs: number }>;   // L1..L8 for this chain
  narrative: string[];      // deterministic lines as in §1, generated server-side from the records
  completeness: { ok: boolean; missing: string[] };        // e.g. ['market.events:<id> (expired by retention; snapshot used)']
}
```

`narrative` is produced by a deterministic formatter (unit-tested), **not** by an LLM. AI explanations, if any, are returned separately under `ai`.

## 5. Explanation templates

- Templates live in `modules/strategy/domain/explanations.ts`, keyed by reason code, with typed parameters.
- Example: `WAVE_CONFIRMED` → `"Wave score {score} ≥ threshold {threshold} with {smartCount} smart wallets and {followBuyers} follow-on buyers."`
- Each reason code has exactly one template. A test asserts every code in the registry has a template and every template parameter is supplied.

## 6. Reason code registry

Codes are stable strings defined in `@paperbot/core/errors/codes.ts`:

| Family | Codes |
|---|---|
| Decision | `WAVE_CONFIRMED`, `ENTRIES_PAUSED`, `DATA_STALE`, `SIGNAL_EXPIRED`, `PRICE_DRIFT`, `ALREADY_IN_POSITION`, `AI_VETO`, `AI_UNAVAILABLE`, `AWAITING_AI`, `RISK_REJECTED`, `EXIT_TAKE_PROFIT`, `EXIT_SCALE_OUT`, `EXIT_STOP_LOSS`, `EXIT_TRAILING_STOP`, `EXIT_TIME`, `EXIT_SMART`, `EXIT_LIQUIDITY`, `EXIT_MANUAL`, `EXIT_RUN_STOP` |
| Wave gates | `G_MIN_SMART`, `G_MIN_FOLLOW`, `G_LIQUIDITY`, `G_EXTENSION`, `G_NOT_DUMPING`, `G_SMART_NOT_EXITING`, `G_DATA_FRESH`, `G_TOKEN_AGE`, `G_POOL_SUPPORTED` |
| Wave terminal | `WAVE_EXPIRED`, `WAVE_INVALIDATED_SCORE`, `WAVE_LIQUIDITY_COLLAPSE`, `WAVE_SMART_SELLING`, `WAVE_DRAWDOWN`, `WAVE_TOKEN_UNSAFE`, `NO_SUPPORTED_POOL` |
| Risk | all `RSK_*` codes in [15](15-risk-engine-spec.md) |
| Execution | `STALE_MARKET_DATA`, `SLIPPAGE_EXCEEDED`, `INSUFFICIENT_LIQUIDITY`, `SIMULATED_TX_FAILURE`, `POOL_UNAVAILABLE`, `INSUFFICIENT_FUNDS`, `EXPIRED`, `ENGINE_RESTART`, `UNSUPPORTED_POOL_TYPE` |
| Safety | `SAF_MODE_NOT_ALLOWED`, `SAF_LIVE_DISABLED`, `SAF_FORBIDDEN_ENV`, `SAF_RPC_METHOD_BLOCKED` |

## 7. Completeness checking

- A job (`pnpm trace:audit --run <id>`, also run in CI on golden scenarios and daily in production) walks every decision and asserts the chain is complete. The only allowed gap is expired raw events, and then only when the signal's feature snapshot is present.
- Metric: `trace_completeness_ratio` (target 1.0). Below 1.0 → WARN.

## 8. Log correlation

Every structured log line on the decision path carries `runId`, `waveId`, `signalId`, `decisionId`, `orderId`, `positionId` as they become known ([22](22-observability-spec.md) §2). This lets logs be joined with the DB trace.
