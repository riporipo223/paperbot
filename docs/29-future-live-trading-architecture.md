# 29 — Future Live Trading Architecture (NOT IMPLEMENTED)

| Field | Value |
|---|---|
| Status | Draft v0.1. **Informational. Nothing in this document may be implemented in the current project stage.** |
| Upstream | [05](05-system-architecture.md), [16](16-paper-trading-spec.md), [21](21-security-spec.md) |
| Downstream | A future project stage (separate approval, separate security review) |
| Used by phases | 26 (shadow prep only reads §3) |

## 1. Why this document exists

To make sure the MVP architecture does not paint us into a corner. It describes how SHADOW and LIVE would attach **without** rewriting the pipeline, and it lists the hard preconditions for LIVE. It is a boundary, not a plan to implement.

## 2. Progression

```text
SIMULATION (MVP)  →  SHADOW (Phase 26 prep; later full)  →  LIVE (future stage, separate approval)
paper fills          paper fills + real read-only quotes     real signed transactions via isolated signer
```

Gate to move from SIMULATION to SHADOW: Phase 25 validation report accepted by the operator.
Gate to move from SHADOW to LIVE (future): the criteria in §5, all met and signed off.

## 3. SHADOW mode (the only part touched in this project stage, Phase 26 preparation)

- `ShadowExecutor` wraps `PaperExecutor`. For each order it additionally requests a **read-only quote** from an aggregator (e.g. the Jupiter quote API) for the same input amount at decision time, and again at the simulated execution time. It records the comparison in a `sim.shadow_quotes` table (Phase 26 migration).
- It measures the divergence between the paper model and the real routed quote (price, route, price impact, fees).
- **No transaction is built, signed or sent.** Quote endpoints are added to `security.allowed_hosts` explicitly. Swap/transaction-building endpoints stay blocked by the host/path allowlist.

## 4. LIVE architecture sketch (future)

```text
Engine (unchanged pipeline) ──ApprovedOrder──▶ RealExecutor (thin client, no keys)
                                                   │ mTLS, signed request, idempotency key
                                                   ▼
                                     Signer Service (separate host/account)
                                     ├─ independent policy engine (per-tx max, daily max, program allowlist,
                                     │  token account allowlist, destination = self only, rate limits)
                                     ├─ key in KMS/HSM or remote signer; never exported
                                     ├─ builds swap tx via aggregator/DEX SDK, simulates, signs, sends
                                     └─ reports signature + landing status back
Engine ◀── execution reports (landed/failed, actual amounts from chain) ── Chain confirmation watcher
Portfolio: ledger entries from ON-CHAIN confirmed balances (reconciliation against wallet state)
```

Properties:
- The engine never holds keys, even in LIVE. Compromising the engine cannot bypass the signer's independent limits.
- The same `TradingExecutor` interface; the same decision/risk pipeline; the same ledger (with a `LIVE` account type and on-chain reconciliation).
- A real kill switch at the signer (deny all) that is independent of the engine.

## 5. Preconditions for any LIVE work (future stage)

1. Validation report shows positive expectancy after costs across the required sensitivity set ([19](19-backtesting-spec.md) §7), on held-out data, with ≥ 200 paper trades.
2. Shadow mode run ≥ 2 weeks, with the paper-vs-real quote divergence within agreed bounds.
3. Independent security review of the signer design. Threat model updated.
4. Written limits: max per trade, max per day, max total capital at risk (tiny initial amounts).
5. Legal/regulatory check for the operator's jurisdiction.
6. Explicit, written operator approval to begin the LIVE project stage.

## 6. What the MVP already does to make this possible

- `TradingExecutor` abstraction and executor factory by mode.
- Write-ahead order records and idempotency keys (required for exactly-once submission).
- Economics output fields map 1:1 to what a real fill report contains.
- The ledger supports additional account types.
- Latency and quality instrumentation is already in place.

## 7. What the MVP must NOT do

Include signing libraries, keypair handling, transaction building, send methods, "test" live paths behind flags, or any LIVE-mode code other than the throwing stub. These are enforced by [21](21-security-spec.md) §2.
