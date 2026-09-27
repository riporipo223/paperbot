# Phase 11 — Economics Engine

| Field | Value |
|---|---|
| Milestone | M3 Headless paper trading |
| Depends on | 01, 07 (venue fixtures and verified fee models via 06) |
| Size | M |
| Requirements | FR-ECO-001, FR-EXE-005/006 (model side) |

## 1. Objective
Implement the pure economics module: swap quotes per venue fee model, max input for an impact limit, network/priority/tip costs, ATA rent handling, the latency sampler, the failure model, the MEV model, and `resolveExecution`, which produces a complete, reproducible fill or failure breakdown. No I/O.

## 2. Context (read first)
- [17](../docs/17-economics-engine-spec.md) (entire)
- [10](../docs/10-market-data-spec.md) §5
- [16](../docs/16-paper-trading-spec.md) §5–7 (how the executor uses it)
- `fixtures/venues/*/README.md` (real trades with observed outputs)

## 3. Dependencies
Phase 01 DONE. Phase 07 DONE (decoders and fixtures). Verified fee models in `ref.venues` (Phase 06).

## 4. Inputs
Core money primitives, the fee model schema, decoded pool reserves types, and venue fixtures.

## 5. Outputs
`modules/economics` (pure): `quoteSwap`, `maxInputForImpact`, `networkCost`, `sampleLatency`, `drawFailure`, `applyMev`, `resolveExecution`, `estimateRoundTripCostRatio`, and types `SwapQuote`, `ExecutionResult`, `CostBreakdown`. Model version constant `economics@1.0.0`.

## 6. Files To Create
```text
apps/engine/src/modules/economics/
  index.ts
  domain/{cp-amm.ts,virtual-curve.ts,fees.ts,impact.ts,network-cost.ts,latency.ts,failure.ts,mev.ts,resolve-execution.ts,cost-ratio.ts,types.ts,inverse-normal.ts}
  __tests__/{cp-amm.test.ts,virtual-curve.test.ts,golden-venues.test.ts,impact.test.ts,latency.test.ts,failure.test.ts,mev.test.ts,resolve-execution.test.ts,properties.test.ts}
```

## 7. Files To Modify
None outside the module (dependency-cruiser: economics may import only `@paperbot/core`).

## 8. Database Changes
None.

## 9. API Changes
None.

## 10. Environment Variables
None.

## 11. Implementation Tasks

**11.1 CP AMM math with input fee**
- Tests first: known small-number examples computed by hand. Floor rounding. Zero/negative input rejected. Output < reserve_out always.

**11.2 Virtual curve math**
- Tests first: buy and sell per [17](../docs/17-economics-engine-spec.md) §4.2 with fee placement per the verified program behaviour. Clamping to real reserves (per the verified behaviour: failure or clamp). Complete curve → `POOL_UNAVAILABLE` for buys.

**11.3 Golden venue tests**
- Tests first: for each verified venue, ≥ 5 recorded real trades (inputs + pre-trade reserves from fixtures) reproduce the observed outputs within ±1 raw unit. **If a venue fails this, it must not be marked supported.** Record the result and update DP-04.

**11.4 Impact and max input**
- Tests first: `price_impact_bps` definitions ([17](../docs/17-economics-engine-spec.md) §4.4). `maxInputForImpact` vs brute-force binary search within 1 unit.

**11.5 Network cost**
- Tests first: base × signatures + priority (`cu_price × cu_limit / 1e6`, floor) + tip.

**11.6 Latency sampler**
- Tests first: inverse-normal accuracy (vs known quantiles). With 100k draws from a seeded RNG, median and p95 are within ±2% of config. Min/max clamps. Determinism for a fixed `u`.

**11.7 Failure model ordering**
- Tests first (table-driven): STALE > POOL_UNAVAILABLE > INSUFFICIENT_LIQUIDITY > SIMULATED_TX_FAILURE > SLIPPAGE_EXCEEDED > FILLED. The fees charged per cause match [17](../docs/17-economics-engine-spec.md) §6.

**11.8 MEV model**
- Tests first: probability gating by `u`. Degradation bounded by the tolerance. Never below `min_out` when filled. `fail_instead` path → SLIPPAGE_EXCEEDED.

**11.9 resolveExecution**
- Tests first: full breakdown fields (three mids, exec price, impact, adverse move, slippage, each fee, rent delta, total cost USD) for BUY with a new ATA, SELL with a full exit + ATA close, SELL partial, and failed cases. Deterministic for fixed inputs.

**11.10 Cost ratio estimate**
- Tests first: matches [17](../docs/17-economics-engine-spec.md) §8 on an example. At $2 size with default priority fees, the ratio is computed and printed in the test output for documentation.

**11.11 Property tests**
- Monotonicity of output in input. Buy exec ≥ mid. Sell exec ≤ mid. Fees ≥ 0. k non-decreasing post-swap with fees. Breakdown components sum to the totals.

## 12. Acceptance Criteria
1. Golden venue tests pass for every venue marked supported. Unsupported venues are documented.
2. All property tests pass (≥ 1,000 runs each).
3. The module has zero imports outside `@paperbot/core` (depcruise).
4. Coverage ≥ 95% lines, mutation score ≥ 70% (Stryker, advisory report).

## 13. Tests
Unit, property, and golden. No integration tests.

## 14. Failure Cases
All handled as `ExecutionResult` failures. There are no exceptions for domain outcomes. Invalid inputs (negative amounts, zero reserves) → `ECO_INVALID_INPUT` result errors.

## 15. Observability
None (pure). The callers log the breakdowns.

## 16. Security
None specific.

## 17. Verification
```bash
pnpm test --filter @paperbot/engine -- economics
pnpm stryker run --mutate "apps/engine/src/modules/economics/domain/**"   # advisory
```

## 18. Commit Strategy
1. `feat(economics): add cp amm and virtual curve swap math`
2. `test(economics): add golden venue trade reproduction`
3. `feat(economics): add impact, network cost and cost ratio`
4. `feat(economics): add latency, failure and mev models`
5. `feat(economics): add resolveExecution with full breakdown`

## 19. Definition Of Done
- [ ] Acceptance criteria met. Global DoD satisfied.
- [ ] DP-04 reflects the golden test results.
