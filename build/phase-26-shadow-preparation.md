# Phase 26 — Shadow Trading Preparation

| Field | Value |
|---|---|
| Milestone | M8 Shadow-ready |
| Depends on | 25 (verdict report accepted by the operator) |
| Size | M |
| Requirements | FR-EXE-001 (`ShadowExecutor`), FR-SAF-001..005 (still enforced) |

## 1. Objective
Enable `SHADOW` mode: the paper executor remains the source of fills, and additionally, for each order, a **read-only** real-world aggregator quote is fetched and recorded at decision time and at simulated execution time. The divergence between the paper economics and the real routing is measured. **No transaction is built, signed or sent. No keys.**

## 2. Context (read first)
- [29](../docs/29-future-live-trading-architecture.md) §3, §6, §7
- [16](../docs/16-paper-trading-spec.md) §2
- [21](../docs/21-security-spec.md) §2 (host allowlist), §9
- [27](../docs/27-data-provider-reference.md) (add the aggregator quote API facts after verification)

## 3. Dependencies
Phase 25 DONE, and the operator explicitly approves proceeding to SHADOW preparation (recorded in [32](../docs/32-architecture-decision-records.md) as a new ADR).

## 4. Inputs
The PaperExecutor, the economics breakdowns, and the validation findings.

## 5. Outputs
- A new ADR: "Shadow mode quote source" (e.g. Jupiter quote API, read-only), with verified limits/terms in [27](../docs/27-data-provider-reference.md).
- Migration `0014_shadow_quotes.sql` (`sim.shadow_quotes`: order_id, phase DECISION|EXECUTION, requested_at, responded_at, in/out amounts, route summary, price impact, fees reported, provider, raw excerpt, status).
- `ShadowExecutor` implementation (delegates to PaperExecutor + a quote recorder). `features.shadow_enabled` flag. Config schema allows `run.mode = SHADOW` only when the flag is on.
- A quote client restricted to the **quote endpoint path** (a host + path allowlist). Swap/transaction-building paths are explicitly blocked (test).
- Dashboard: a divergence panel on the order detail and a Performance section "paper vs quoted" (distribution of divergence bps).
- A divergence analytics report.

## 6. Files To Create
```text
packages/db/migrations/0014_shadow_quotes.sql
apps/engine/src/modules/execution/shadow/{quote-client.ts,quote-recorder.ts,divergence.ts}
apps/engine/src/modules/execution/__tests__/{shadow-executor.test.ts,quote-client.test.ts,divergence.test.ts}
apps/dashboard/components/orders/shadow-divergence.tsx
apps/dashboard/components/performance/paper-vs-quoted.tsx
fixtures/providers/<aggregator>/*.json
```

## 7. Files To Modify
- `modules/execution/shadow-executor.ts` (the implementation replaces the stub body)
- `modules/execution/executor-factory.ts` (SHADOW when the flag is on)
- `packages/config` schema (SHADOW allowed with the flag) + [26](../docs/26-configuration-reference.md)
- `security.allowed_hosts` (+ a path allowlist structure) + [21](../docs/21-security-spec.md)
- [16](../docs/16-paper-trading-spec.md) §2, [29](../docs/29-future-live-trading-architecture.md) §3 (status)

## 8. Database Changes
`0014_shadow_quotes.sql`. The `strategy.runs.mode` CHECK already allows `SHADOW`.

## 9. API Changes
- `GET /api/v1/orders/:orderId` includes `shadowQuotes`.
- `GET /api/v1/runs/:runId/analytics/shadow-divergence`.
Update [08](../docs/08-api-spec.md).

## 10. Environment Variables
An aggregator API key only if the verified provider requires one (read-only quote access). The forbidden-env guard is unchanged.

## 11. Implementation Tasks

**26.1 Verify the quote provider (docs + probe) and record the ADR**
- Official docs only. Record the endpoint, limits, terms, and fields. Record fixtures.

**26.2 Path-level allowlist**
- Tests first: requests to the quote path succeed (mock). Any swap/transaction path (e.g. `/swap`, `/swap-instructions`, or any path not allowlisted) throws `SAF_PATH_NOT_ALLOWED` before network I/O.

**26.3 Quote client + recorder**
- Tests first (recorded fixtures): parse the quote into the normalized shape. Timeouts. Rate limits. Recording rows at the DECISION and EXECUTION phases. Quote failures never affect the paper fill.

**26.4 ShadowExecutor**
- Tests first: delegates `submit`/`execute` to the PaperExecutor with identical fill results (property: shadow on/off produce identical ledgers). Quotes are recorded asynchronously.

**26.5 Divergence analytics**
- Tests first: bps divergence between the paper `amount_out` and the quoted `outAmount` for the same input at the same phase. Distribution stats. Attribution (route vs impact vs fees where available).

**26.6 Config + factory enablement**
- Tests first: SHADOW is refused without the flag. Allowed with it. LIVE is still refused in all layers (regression tests).

**26.7 Dashboard panels**
- Component tests + an E2E check with fixture quotes.

**26.8 Shadow run (≥ 7 days) + report**
- Record the divergence results in `reports/shadow/<date>.md`, with recommendations to calibrate the economics model (new config versions).

## 12. Acceptance Criteria
1. SHADOW runs record real quotes without affecting the paper fills (property test + production check).
2. Transaction-building/sending is impossible (path allowlist tests, safety scan, no signing libraries).
3. The divergence report is produced.
4. All LIVE blocks remain intact (regression tests).

## 13. Tests
Unit, contract (fixtures), property (ledger equality), and E2E (panel).

## 14. Failure Cases
- Quote provider down/rate-limited → quotes are recorded as `ERROR`/`RATE_LIMITED`, and the paper trading continues unaffected.

## 15. Observability
Metrics: quote latency, error rate, and a divergence histogram.

## 16. Security
The highest-risk phase for scope creep. Only quote endpoints are allowed. Any request to add transaction building or signing is **out of scope** and requires a separate project stage ([29](../docs/29-future-live-trading-architecture.md) §5).

## 17. Verification
```bash
pnpm test --filter @paperbot/engine -- shadow executor-factory
pnpm check:safety
```
Plus the shadow report.

## 18. Commit Strategy
1. `docs: verify shadow quote provider and add adr`
2. `feat(db): add shadow quotes table`
3. `feat(security): add path-level egress allowlist`
4. `feat(execution): add read-only quote client and recorder`
5. `feat(execution): implement shadow executor`
6. `feat(dashboard): add shadow divergence panels`
7. `docs: shadow run report`

## 19. Definition Of Done
- [ ] Acceptance criteria met. Global DoD satisfied.
- [ ] Milestone M8 marked. LIVE remains unimplemented and blocked.
