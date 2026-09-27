# Phase 06 — Reference Data: Venues, Tokens, Pools

| Field | Value |
|---|---|
| Milestone | M1 Truthful data |
| Depends on | 04 (and 03 fixtures) |
| Size | M |
| Requirements | FR-REF-001, FR-REF-002, FR-REF-003 |

## 1. Objective
Resolve and persist token metadata and safety facts, discover a token's pools from DexScreener/GeckoTerminal and on-chain data, maintain the venue registry (with verified fee models and a `supported` flag), and select a primary tradable pool per token, or mark the token unsupported.

## 2. Context (read first)
- [10](../docs/10-market-data-spec.md) §4, §7, §10
- [07](../docs/07-database-schema.md) §3.1
- [17](../docs/17-economics-engine-spec.md) §3 (fee model schema)
- [27](../docs/27-data-provider-reference.md) §5–6 (verified)
- ADR-0010

## 3. Dependencies
Phase 04 DONE. Phase 03 fixtures available.

## 4. Inputs
Pipeline, RPC client, DexScreener/GeckoTerminal HTTP clients + schemas, and the verified venue list (DP-04).

## 5. Outputs
- Migration `0006_ref.sql` with the venue seed rows (`supported=false` unless verified in Phase 03; verified ones set `supported=true` with `verified_at` and `verification_ref`).
- `modules/reference`: `TokenResolver` (mint account → decimals, program, authorities, extensions, supply), `SafetyObserver` (periodic refresh, top-10 holders optional), `PoolDiscovery` (DexScreener tokens endpoint + GeckoTerminal fallback), `VenueRegistry`, `PrimaryPoolSelector`, repositories, and events `token.discovered`, `pool.discovered`, `pool.selected`, `token.rejected`.
- A public query API: `getToken`, `getPools(mint)`, `getPrimaryPool(mint)`, `getSafety(mint)`, `getVenue(id)`.

## 6. Files To Create
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

## 7. Files To Modify
- `packages/core/src/events/event-types.ts` (register `token.discovered`, `pool.discovered` v1)
- Wallet swap flow: on `wallet.swap.detected` with an unknown mint → request resolution (reference subscribes to the bus)

## 8. Database Changes
`0006_ref.sql`: `ref.venues`, `ref.tokens`, `ref.token_safety_observations`, `ref.pools` + seed venues.

## 9. API Changes
None (internal query API only).

## 10. Environment Variables
None new.

## 11. Implementation Tasks

**06.1 Venue registry + fee model validation**
- Tests first: seed rows load. The fee model validates against the zod schema. `supported=true` requires `verified_at` and `verification_ref` (a DB CHECK + app validation). An unknown program ID → `OTHER`/unsupported.

**06.2 TokenResolver**
- Tests first, with a recorded `getAccountInfo` jsonParsed for SPL and Token-2022 mints: decimals, program, authorities, and extensions are extracted. Invalid mint → `REF_INVALID_MINT`. The metadata strings (symbol/name/uri) are treated as untrusted: stored raw but truncated (64/64/200) and stripped of control characters.
- Emits `token.discovered` (persisted event) and upserts `ref.tokens`.

**06.3 SafetyObserver**
- Tests first: an observation is created with authorities and extensions. Refresh happens every `reference.safety_refresh_minutes` while watched (SimulatedClock). The optional `getTokenLargestAccounts` top-10 excludes known pool vault accounts. A stale observation is flagged when its age exceeds `safety_max_age_minutes`.

**06.4 Pool discovery**
- Tests first with recorded DexScreener/GeckoTerminal payloads: pools are mapped to `ref.pools` with the venue ID resolved via `dexId`/program mapping (mapping table verified in Phase 03), base/quote orientation normalized (base = the token, quote = SOL/USDC), and vaults/state accounts resolved on-chain when the provider lacks them (`getAccountInfo` on the pool account with the venue decoder's account parser; for the Phase 07 decoders, a minimal account-field reader is enough here).
- Emits `pool.discovered`. Respects the rate limits (DexScreener pairs 300/min × safety factor).

**06.5 PrimaryPoolSelector**
- Tests first (pure, table-driven): chooses the supported, active, SOL-quoted pool with the highest liquidity. A completed pump curve → the migrated pool. None supported but unsupported exist → `UNSUPPORTED_POOL_TYPE`. None at all → `NO_POOL`. Re-selection on new pool discovery. Migration while a position is open emits `pool.selected{reason:'MIGRATION'}`.

**06.6 Resolution orchestration**
- Tests first: an unknown mint from a wallet swap triggers resolve token → discover pools → select primary → emit `pool.selected` or `token.rejected{reason}` within `wave.pool_resolution_timeout_s` (it records a timeout otherwise). Concurrent requests for the same mint are coalesced (single flight).

## 12. Acceptance Criteria
1. For real tokens bought by tracked wallets, the engine persists token, safety and pools, and selects a primary pool (spot-check ≥ 10 tokens; record results).
2. Unsupported pool types are rejected with a reason, never approximated.
3. Venue `supported` flags reflect Phase 03 verification only.
4. Resolution is single-flight and time-bounded.

## 13. Tests
Unit (selector, safety, fee model), contract (provider payload mapping), integration (repositories, resolution flow against recorded payloads via a mock agent).

## 14. Failure Cases
- DexScreener down → GeckoTerminal fallback (rate-limited). Both down → on-chain-only discovery is not possible in the MVP → `token.rejected{NO_POOL_DATA}` (retry later if the wave is still active).
- Token-2022 with forbidden extensions → the observation records them. Risk (Phase 13) rejects later. Reference does not decide tradability except on pool type.

## 15. Observability
Log `token.resolved`, `pool.selected`, `token.rejected` with reasons. Metrics: resolution latency and rejections by reason.

## 16. Security
Untrusted metadata is sanitized at ingest. It is never used in logs without truncation.

## 17. Verification
```bash
pnpm test --filter @paperbot/engine -- reference
pnpm test:integration --filter @paperbot/engine -- reference
```
Plus the manual spot-check table in the verification note.

## 18. Commit Strategy
1. `feat(db): add reference tables and venue seeds`
2. `feat(reference): add token resolver and safety observer`
3. `feat(reference): add pool discovery from dexscreener and geckoterminal`
4. `feat(reference): add primary pool selection and resolution flow`

## 19. Definition Of Done
- [ ] Acceptance criteria met. Global DoD satisfied.
