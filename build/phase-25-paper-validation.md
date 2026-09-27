# Phase 25 — Paper Trading Validation

| Field | Value |
|---|---|
| Milestone | M7 Evidence |
| Depends on | 24 |
| Size | L (calendar time: ≥ 14 days of continuous running recommended) |
| Requirements | Success metrics [01](../docs/01-product-spec.md) §7, FR-RPL-003/004, [22](../docs/22-observability-spec.md) §8 |

## 1. Objective
Run the system continuously in SIMULATION on real data, establish measured latency baselines (replacing the provisional ceilings), verify operational metrics (traceability, invariants, determinism, uptime), perform out-of-sample parameter and sensitivity studies via replay, and write an evidence-based strategy verdict report. **No code changes to strategy logic during a measurement window.** Fixes create new strategy versions and restart the window.

## 2. Context (read first)
- [01](../docs/01-product-spec.md) §7
- [19](../docs/19-backtesting-spec.md) §2, §7
- [18](../docs/18-portfolio-accounting-spec.md) §7
- [17](../docs/17-economics-engine-spec.md) §9 (limitations)
- [22](../docs/22-observability-spec.md) §8
- [11](../docs/11-wallet-intelligence-spec.md) §8 (out-of-sample wallets)

## 3. Dependencies
Phase 24 DONE. The operator supplies the seed wallet list (DP-05) and decides the AI mode (DP-06).

## 4. Inputs
The production paper environment.

## 5. Outputs
- `reports/validation/<date>-baselines.md`: latency baselines per stage (p50/p95/p99) on the production host.
- [22](../docs/22-observability-spec.md) §8 table filled in. `observability.latency_budgets` set (engine config).
- [03](../docs/03-system-requirements.md) §2 annotated: "superseded by measured baselines (link)".
- `reports/validation/<date>-strategy-verdict.md`: the full report (template in §11.6).
- Recommendations: next strategy version and config changes (as new versions, not edits), and a Helius plan decision (DP-07).
- Optional: `research/` notebooks for analysis (Python allowed here; not imported by the runtime).

## 6. Files To Create
```text
reports/validation/<date>-baselines.md
reports/validation/<date>-strategy-verdict.md
reports/validation/<date>-sensitivity/*.md       (study outputs)
config/strategies/smart-money-wave/<next>.yaml   (only if recommended; new file)
```

## 7. Files To Modify
- [22](../docs/22-observability-spec.md) §8, [03](../docs/03-system-requirements.md) §2 (annotation), `config/base.yaml` (`observability.latency_budgets`)
- [32](../docs/32-architecture-decision-records.md) DP-06/DP-07 outcomes
- [17](../docs/17-economics-engine-spec.md) §9 (any new limitation discovered)

## 8. Database Changes
None.

## 9. API Changes
None.

## 10. Environment Variables
None.

## 11. Implementation Tasks

**25.1 Measurement plan (before starting)**
- Write down the window length, the config version, the wallet list version (hash of the seed file), the capture mode (recommend `BROAD` for part of the window, to reduce the replay coverage bias), the scoring cut-off date (out-of-sample), and the AI mode.

**25.2 Continuous run + daily checks**
- Daily: trace audit = 1.0, zero invariant violations, dead letters within the baseline, uptime, conserve-mode share, and the provider health summary. Log them in the report's daily table.

**25.3 Latency baselines (after ≥ 72 h)**
- Extract the percentiles from `ops.latency_samples`. Set the budgets = p95 × 1.5. Update the docs/config.

**25.4 Determinism check**
- `pnpm replay:verify` on at least one full production day. Record the result.

**25.5 Studies (replay)**
- Declare the train/held-out ranges. Run the required sensitivity preset ([19](../docs/19-backtesting-spec.md) §7) and a small parameter grid (≤ 20 configurations, reported). Compare `AS_RECEIVED` vs `BY_EVENT_TIME` to quantify the latency cost.

**25.6 Verdict report (template)**
1. Setup (versions, window, wallets, capture mode, AI mode, host).
2. Operational health (uptime, gaps, conserve mode, errors).
3. Latency baselines.
4. Trading results (all [18](../docs/18-portfolio-accounting-spec.md) §7 metrics; SOL and USD; the sample-size caveat).
5. Cost decomposition (fees, network, priority, impact, adverse move, MEV, rent lock).
6. Rejection analysis (by reason code. What would have happened: shadow outcomes of rejected signals computed via replay counterfactual configs).
7. Sensitivity results (latency ×, failure rate, MEV, slippage, priority fees).
8. Held-out performance vs training.
9. Model limitations and threats to validity.
10. Verdict: **positive expectancy after costs: yes / no / inconclusive**, with a confidence discussion.
11. Recommendations (next versions, data upgrades, whether to proceed to SHADOW).

## 12. Acceptance Criteria
1. ≥ 7 consecutive days of continuous engine uptime with automatic recovery from provider disconnects (NFR-REL-001). ≥ 14 days recommended.
2. Trace completeness 1.0. Zero ledger invariant violations. Zero fills on non-FRESH data (a query-based check in the report).
3. Baselines recorded and budgets configured.
4. The replay determinism check passes on a production day.
5. The verdict report is complete, including limitations and held-out results.

## 13. Tests
The verification queries are committed as `scripts/validation-queries.sql` (e.g. fills joined with pool state age > fill threshold must return 0 rows).

## 14. Failure Cases
- The strategy produces too few trades for significance → the verdict is "inconclusive". Recommend extending the window or broadening capture/wallets (as new versions).
- An operational issue discovered → fix (new code version) and restart the measurement window. Record it in the report.

## 15. Observability
The report relies on stored metrics. Screenshots of the Performance/System pages are attached.

## 16. Security
Re-run the secret/safety scans on the deployed image version. Confirm no LIVE code paths (the scan output is included in the report).

## 17. Verification
The report itself plus `scripts/validation-queries.sql` outputs.

## 18. Commit Strategy
1. `docs: add validation measurement plan`
2. `docs: record latency baselines and budgets`
3. `docs: add sensitivity study reports`
4. `docs: add strategy verdict report`
5. `feat(config): add smart-money-wave <next> config` (if recommended)

## 19. Definition Of Done
- [ ] Acceptance criteria met. Global DoD satisfied.
- [ ] Operator has reviewed the verdict report. Milestone M7 marked.
