# Phase 03 — Provider Verification & Client Foundation

| Field | Value |
|---|---|
| Milestone | M1 Truthful data |
| Depends on | 01 |
| Size | M |
| Requirements | FR-ING-008, FR-SAF-005, [27](../docs/27-data-provider-reference.md) §1 policy |

## 1. Objective
Replace provisional provider facts with verified ones, capture real fixtures, and build the generic, safe client foundation: HTTP client with host allowlist, token-bucket rate limiter, retry/circuit breaker, Helius credit budget, the read-only Solana RPC client, and a generic reconnecting WebSocket client. **No normalization or pipeline integration yet.**

## 2. Context (read first)
- [27](../docs/27-data-provider-reference.md) (entire; §10 is this phase's checklist)
- [09](../docs/09-real-time-data-architecture.md) §11–13
- [10](../docs/10-market-data-spec.md) §2, §4
- [21](../docs/21-security-spec.md) §2 (RPC allowlist, host allowlist)
- ADR-0009, ADR-0015

## 3. Dependencies
Phase 01 DONE. (Phase 02 is not required. Credit usage is persisted from Phase 04 on.)

## 4. Inputs
Core primitives and config. The operator provides `HELIUS_API_KEY` (and optionally `COINGECKO_API_KEY`) in the local environment for recording. **If keys are unavailable, the agent must stop after Tasks 03.5–03.9 (which need no network) and report the blocker. It must not fabricate fixtures.**

## 5. Outputs
- Updated [27](../docs/27-data-provider-reference.md) with a `DOC-VERIFIED`/`PROBE-VERIFIED` status per row, corrected values, and the change log.
- Updated defaults in `config/base.yaml` for verified limits (e.g. `max_subscriptions_per_connection`).
- `fixtures/providers/{helius,dexscreener,geckoterminal,coingecko}/…` and `fixtures/venues/{pumpfun_curve,pumpswap,raydium_amm_v4,raydium_cpmm}/…` (account data, logs notifications, parsed transactions), each with a capture README.
- Recorder scripts under `scripts/recorders/`.
- Client foundation in `apps/engine/src/modules/ingestion/shared/`.
- An ADR-0009 measurement plan (the latency comparison itself is done in Phase 05).

## 6. Files To Create
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

## 7. Files To Modify
- `docs/27-data-provider-reference.md` (verification columns, facts, change log)
- `docs/10-market-data-spec.md` §4 (confirmed program IDs/layout notes), `docs/17-economics-engine-spec.md` §3 (verified fee facts), if they differ
- `config/base.yaml` (verified limits, allowed hosts)
- `docs/32-architecture-decision-records.md` (ADR-0009 notes, DP-04 venue list status)

## 8. Database Changes
None.

## 9. API Changes
None.

## 10. Environment Variables
`HELIUS_API_KEY` (required for recording), `COINGECKO_API_KEY` (optional).

## 11. Implementation Tasks

**03.1 Documentation verification (no code)**
- Read each official source listed in [27](../docs/27-data-provider-reference.md) §4–7. Update every row's `Verification`. Resolve the flagged conflicts (Helius Enhanced WS tier, CoinGecko Demo limit, streaming credit applicability). Record the URLs and dates.
- Done when: no row remains `SEARCH-SNIPPET`/`UNVERIFIED` without an explicit "could not verify because…" note.

**03.2 Recorder scripts**
- Implement small CLI scripts that call real endpoints and write sanitized JSON (strip the API key from URLs/headers). For Helius WS: subscribe to `logsSubscribe({mentions:[<active pump.fun token mint or pool>]})` for N minutes, and `accountSubscribe` on a bonding curve and on a PumpSwap/Raydium pool account.
- Recorders are **never** run by tests.

**03.3 Record provider fixtures**
- For each provider: at least one success payload per endpoint used, one 429 (if safely obtainable, otherwise documented), and one error shape.
- Probe the DexScreener/GeckoTerminal update cadence (poll one active pair every 2 s for 10 min) and record the results in [27](../docs/27-data-provider-reference.md).

**03.4 Record venue fixtures**
- For each candidate venue: ≥ 5 account data samples with slots, ≥ 20 trade log notifications, and ≥ 5 full transactions (`getTransaction` jsonParsed). Include known outputs (amounts in/out) for economics golden tests.
- Record the program IDs and the IDL/source reference used for layouts (commit hash or URL) in `fixtures/venues/<venue>/README.md`.

**03.5 HTTP client with host allowlist**
- Tests first: a request to a host not in `security.allowed_hosts` throws `SAF_HOST_NOT_ALLOWED`. Timeouts abort. The API key query param is redacted in logged URLs. Responses are returned as `unknown` (the caller validates).

**03.6 Token-bucket rate limiter**
- Tests first (SimulatedClock): N requests per window allowed, the next waits. Concurrent acquirers are FIFO. `safety_factor` is applied. `Retry-After` pauses the bucket.

**03.7 Retry + circuit breaker**
- Tests first: retries only on retryable errors (network/5xx/429). Full-jitter bounds. The breaker opens after the threshold, goes half-open after the cooldown, and closes on success.

**03.8 Credit budget**
- Tests first: charging calls by type updates hour/day/month totals. Pace projection. Conserve mode triggers at 90% projected. Budget refusal for low-priority categories while in conserve mode.
- Implement with credit costs from config (verified in 03.1).

**03.9 Read-only Solana RPC client**
- Tests first: every allowlisted method builds the right JSON-RPC body (against recorded responses via a mock agent). `call('sendTransaction')`, `call('requestAirdrop')`, and `call('simulateTransaction')` throw `SAF_RPC_METHOD_BLOCKED` **before** any network I/O. Commitment defaults to `confirmed`.
- Implement a thin wrapper (raw JSON-RPC over the HTTP client. `@solana/kit` may be used only for encoding/decoding helpers, never its transaction/signer APIs; enforced by lint and the safety scan).

**03.10 Reconnecting WS client**
- Tests first, with a local fake WS server: connect → subscribe → receive. Server drop → reconnect with backoff → resubscribe all. Missed pong ×2 → reconnect. Subscription IDs are remapped after reconnect. Inbound backpressure pauses reading at the queue bound. `onStatus` callbacks fire (CONNECTED/DISCONNECTED/RECONNECTING).

**03.11 Contract tests for recorded payloads**
- Tests first: zod schemas for each provider payload used later (DexScreener pair, GeckoTerminal trades/ohlcv/pool, CoinGecko simple price, Helius logs notification, account notification, getTransaction). All recorded fixtures parse. Deliberately corrupted copies fail.
- These schemas live in `modules/ingestion/<provider>/schemas.ts`. This phase creates them, and later phases use them.

## 12. Acceptance Criteria
1. [27](../docs/27-data-provider-reference.md) has verification status for every fact. Conflicts are resolved or explicitly flagged as blockers.
2. Fixtures exist for all providers and candidate venues, with capture READMEs. There are no secrets in fixtures (gitleaks + a manual grep for the key value).
3. The RPC client cannot send transactions (test). HTTP to non-allowlisted hosts is impossible (test).
4. The rate limiter, retry, circuit breaker, credit budget and WS client are unit-tested, including reconnect.
5. Contract tests parse all recorded fixtures.

## 13. Tests
Unit (clients) + contract (fixtures). No live network in tests (the network guard in the Vitest setup blocks sockets except localhost fake servers).

## 14. Failure Cases
- Official doc contradicts the architecture (e.g. Free plan has no usable WS for logs) → **stop and report**. Record the contradiction in ADR-0009/[27](../docs/27-data-provider-reference.md). Do not redesign silently.
- A venue layout can't be verified → keep the venue `supported=false` (DP-04). The decoder for that venue is not implemented in Phase 07.

## 15. Observability
Clients expose hooks for metrics (implemented as no-op interfaces now, wired in Phase 04/22): request counts, latencies, rate-limit waits, breaker state, credits.

## 16. Security
- Fixtures are sanitized. The API key is never written to disk outside `.env.local`.
- Host allowlist and RPC method allowlist enforced by tests.

## 17. Verification
```bash
pnpm test --filter @paperbot/engine -- ingestion/shared
pnpm check:safety
grep -R "$HELIUS_API_KEY" fixtures/ && echo "LEAK" || echo "clean"
```
The verification note includes a table of every provider fact before and after verification.

## 18. Commit Strategy
1. `docs: verify provider facts and update reference`
2. `chore: add provider recorder scripts`
3. `test: add recorded provider and venue fixtures`
4. `feat(ingestion): add http client with host allowlist`
5. `feat(ingestion): add rate limiter, retry and circuit breaker`
6. `feat(ingestion): add helius credit budget`
7. `feat(ingestion): add read-only solana rpc client`
8. `feat(ingestion): add reconnecting websocket client`
9. `test(ingestion): add provider payload contract tests`

## 19. Definition Of Done
- [ ] Acceptance criteria met. Global DoD satisfied.
- [ ] DP-04 updated with the verified venue list.
- [ ] Any blocker is reported to the operator with evidence.
