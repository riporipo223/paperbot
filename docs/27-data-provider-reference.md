# 27 — Data Provider Reference

| Field | Value |
|---|---|
| Status | Draft v0.1. **Provider facts are PROVISIONAL until re-verified in Phase 03.** |
| Upstream | [03](03-system-requirements.md), [09](09-real-time-data-architecture.md), [10](10-market-data-spec.md) |
| Downstream | Phase 03 (verification), ingestion phases 05–08, [26](26-configuration-reference.md) (limits) |
| Used by phases | 03, 05, 06, 07, 08, 09, 24, 26 |

> **Numbering note.** The master prompt refers to this document both as `27-data-provider-reference.md` (§20 list) and as `28-data-provider-reference.md` (§27). The canonical name is **27-data-provider-reference.md**. See [ADR-0019](32-architecture-decision-records.md#adr-0019-document-numbering).

## 1. Verification policy

1. Nothing in this document may be implemented against until the **Phase 03 verification spike** has confirmed it against **official documentation** *and* a live probe (a minimal real API call, recorded as a fixture).
2. Every row carries a `Verification` status:
   - `UNVERIFIED`: assumption or recollection.
   - `SEARCH-SNIPPET`: seen in search-engine results pointing to the provider's official domain, but the page itself could not be fetched. This is what was possible at documentation time: the documentation environment's network proxy blocked `helius.dev`, `apiguide.geckoterminal.com` and `docs.dexscreener.com`.
   - `DOC-VERIFIED (date, URL)`: read on the official page.
   - `PROBE-VERIFIED (date, fixture)`: confirmed by a live call.
3. Blog posts and third-party articles are never sufficient verification.
4. When verification contradicts this document, update this document **first** (in the same commit as the code change) and record the change in §9.

Documentation date: **2026-09-27**.

## 2. Classification legend

`FREE` (no account/key), `FREE TIER` (free plan with limits, key required), `PAID` (requires paid plan), `OPTIONAL` (system works without it), `NOT REQUIRED` (not used in MVP).

## 3. Provider summary

| Provider | Role | MVP classification | Used for fills? |
|---|---|---|---|
| **Helius** | Primary Solana infrastructure: WS streams (logs/account), RPC reads, optional webhooks, wallet history | **FREE TIER** (paid upgrade path) | T1 pool state is the **only** fill source |
| **DexScreener** | Market snapshots, pair/token discovery, cross-check | **FREE** (keyless) | No |
| **GeckoTerminal** | Pool/trade reference, OHLCV history, new pools, fallback snapshots | **FREE** (public API, keyless) | No |
| **CoinGecko** | SOL/USD reference | **OPTIONAL**, FREE TIER (Demo key) | No |
| **Anthropic** | LLM for advisory agents | **OPTIONAL** (PAID per token; `ai.mode=OFF` works) | No |
| **Jupiter** | Future real execution; read-only quotes for shadow comparison | **NOT REQUIRED** (Phase 26 may use the quote API only) | No |
| **Supabase** | Managed Postgres | OPTIONAL (FREE TIER insufficient for event-log volume; see [ADR-0018](32-architecture-decision-records.md#adr-0018-database-hosting)) | n/a |
| **Vercel** | Dashboard hosting | FREE TIER (Hobby) | n/a |

## 4. Helius

Official sources to verify: `https://www.helius.dev/pricing`, `https://www.helius.dev/docs/billing/plans`, `https://www.helius.dev/docs/faqs/websockets`, `https://www.helius.dev/docs/webhooks`, and the RPC method docs.

### 4.1 Plans and limits

| Fact | Value (provisional) | Verification |
|---|---|---|
| Free plan monthly credits | 1,000,000 | SEARCH-SNIPPET (helius.dev pricing/plans) |
| Free plan RPC rate | 10 req/s | SEARCH-SNIPPET |
| Free plan `sendTransaction` rate | 1 req/s (irrelevant: never used) | SEARCH-SNIPPET |
| Free plan DAS & Enhanced API rate | 2 req/s | SEARCH-SNIPPET |
| Free plan `getProgramAccounts` rate | 5 req/s | SEARCH-SNIPPET |
| Free plan concurrent WS connections | 5 | SEARCH-SNIPPET (helius.dev WebSockets FAQ) |
| Standard Solana WS methods (`logsSubscribe`, `accountSubscribe`, `slotSubscribe`, `signatureSubscribe`) on Free | Available | SEARCH-SNIPPET |
| Enhanced WebSockets (`transactionSubscribe` with account filters) | Business plan and above (conflicting snippets; one says Developer has "LaserStream WebSocket features") | SEARCH-SNIPPET, **conflicting → must verify** |
| LaserStream gRPC mainnet | Business / Professional | SEARCH-SNIPPET |
| Developer plan | ~$49/mo, 50 RPC req/s, 150 WS connections | SEARCH-SNIPPET |
| Business plan | ~$499/mo, 200 RPC req/s, 250 WS connections | SEARCH-SNIPPET |
| Professional plan | ~$999/mo, 500 RPC req/s, 1,000 WS connections | SEARCH-SNIPPET |
| Max subscriptions per WS connection | **Unknown** | UNVERIFIED → Phase 03 must determine (docs + probe) |
| Webhooks on Free plan (count, address limit, latency) | Available, charged per event; count/address limits unknown | SEARCH-SNIPPET (partial) → verify |

### 4.2 Credit costs (provisional)

| Operation | Credits | Verification |
|---|---|---|
| Standard RPC call | 1 | SEARCH-SNIPPET |
| `getProgramAccounts`, archival calls | 10 | SEARCH-SNIPPET |
| DAS calls | 10 | SEARCH-SNIPPET |
| Webhook event delivered | 1 (charged even if our endpoint fails) | SEARCH-SNIPPET |
| Streaming data (WebSockets/LaserStream) | 20 credits per MB (reduced 2026-04-07) | SEARCH-SNIPPET. **Must verify whether this applies to standard WS on Free.** |
| Enhanced Transactions API (parsed history) | Unknown | UNVERIFIED |

### 4.3 Budget implications (to be recalculated after verification)

- 1 M credits/month ≈ 33,000/day ≈ 1,390/hour.
- If WS streaming costs 20 credits/MB: a busy memecoin pool at ~100 tx/min × ~2 KB per logs notification ≈ 12 MB/h ≈ 240 credits/h. So roughly **4–5 hot pools continuously** would consume the whole Free budget. Watched pools must be few and short-lived. This is why the watch-set policy ([10](10-market-data-spec.md) §3) and conserve mode ([09](09-real-time-data-architecture.md) §12) exist.
- Wallet backfills (`getSignaturesForAddress` + `getTransaction` per swap) cost ~1–2 credits per historical transaction. They are budgeted per job.
- **Recommended upgrade trigger**: if Phase 25 validation shows conserve mode active > 20% of the time, move to the Developer plan (config change only).

### 4.4 Methods used (read-only allowlist)

`getAccountInfo`, `getMultipleAccounts`, `getTransaction`, `getSignaturesForAddress`, `getSlot`, `getBlockTime`, `getTokenLargestAccounts`, `getTokenSupply`, `getMinimumBalanceForRentExemption`, `getRecentPrioritizationFees`, `getPriorityFeeEstimate` (Helius-specific; verify), `getHealth`, `getVersion`. WS: `logsSubscribe`, `accountSubscribe`, `slotSubscribe` (optional), and the matching `*Unsubscribe`.

**Never used and blocked by the client allowlist:** `sendTransaction`, `sendRawTransaction`, `simulateTransaction` (not needed; blocked to keep the surface minimal), `requestAirdrop`, and anything under Helius "Sender"/transaction submission.

### 4.5 Commitment level

Streams and reads use `confirmed` ([ADR-0015](32-architecture-decision-records.md#adr-0015-commitment-level-confirmed)). Rationale: `processed` can be rolled back, and `finalized` adds ~10+ s of latency. Rollback of `confirmed` is rare. A nightly optional reconciliation can check `finalized` status for signatures used in decisions.

## 5. DexScreener

Official sources to verify: `https://docs.dexscreener.com/api/reference`, `https://docs.dexscreener.com/api/api-terms-and-conditions`.

| Fact | Value (provisional) | Verification |
|---|---|---|
| Authentication | None (keyless) | SEARCH-SNIPPET |
| Pair / token / search endpoints rate limit | 300 requests/min | SEARCH-SNIPPET |
| Token profiles / boosts / community takeovers / ads endpoints | 60 requests/min | SEARCH-SNIPPET |
| Endpoint: pairs by chain + pair address | `GET /latest/dex/pairs/{chainId}/{pairId}` | UNVERIFIED path shape → verify |
| Endpoint: pairs by token address(es) | `GET /tokens/v1/{chainId}/{tokenAddresses}` (comma-separated, up to N) | UNVERIFIED → verify |
| Endpoint: search | `GET /latest/dex/search?q=` | UNVERIFIED → verify |
| Endpoint: latest token profiles | `GET /token-profiles/latest/v1` | UNVERIFIED → verify |
| Fields available per pair | priceUsd, priceNative, liquidity.usd, fdv, marketCap, volume.{m5,h1,h6,h24}, txns.{m5,h1,...}.{buys,sells}, priceChange.*, pairCreatedAt | UNVERIFIED → verify exact names |
| Freshness/caching of pair data | Unknown | UNVERIFIED → Phase 03 probe: poll one active pair every 2 s for 10 min, measure the update cadence |
| Official WebSocket | None documented publicly (assumption) | UNVERIFIED |
| Terms | Must review API Terms & Conditions (attribution, commercial use, caching) | Pending |

Poll plan (MVP): watched tokens batched in one `tokens` call where supported, every `ingestion.dexscreener.poll_ms` (default 10,000), within 80% of the limit.

## 6. GeckoTerminal (public API)

Official sources to verify: `https://apiguide.geckoterminal.com/`, `/faq`, `/changelogs`.

| Fact | Value (provisional) | Verification |
|---|---|---|
| Base URL | `https://api.geckoterminal.com/api/v2` | UNVERIFIED → verify |
| Authentication | None (public) | SEARCH-SNIPPET |
| Rate limit | **30 calls/min** (global, per IP) | SEARCH-SNIPPET (official FAQ) |
| Pool trades endpoint | `/networks/{network}/pools/{pool_address}/trades`: latest **300 trades in past 24 h** | SEARCH-SNIPPET |
| OHLCV endpoint | `/networks/{network}/pools/{pool}/ohlcv/{timeframe}`, `limit` default 100, max 1000, history up to ~6 months | SEARCH-SNIPPET (via client library docs) → verify officially |
| New pools | `/networks/solana/new_pools` | UNVERIFIED → verify |
| Higher limits | Via CoinGecko paid API plans (onchain endpoints) | SEARCH-SNIPPET |
| Data freshness/caching | Unknown (likely cached tens of seconds) | UNVERIFIED → Phase 03 probe |
| Terms/attribution | Must review | Pending |

Budget: 30 calls/min is the scarcest resource. Allocation (`ingestion.geckoterminal.allocation`): 50% trades polling for P0/P1 pools when on-chain trade decoding is unavailable or for gap backfill, 30% OHLCV chart backfill (dashboard on demand, cached), 20% discovery/fallback snapshots.

## 7. CoinGecko (optional)

Official sources to verify: `https://docs.coingecko.com/docs/common-errors-rate-limit`, `https://www.coingecko.com/en/api/pricing`.

| Fact | Value (provisional) | Verification |
|---|---|---|
| Demo (free) plan rate limit | 100 calls/min (some sources say 30 calls/min, so conflicting) | SEARCH-SNIPPET, **conflicting → verify** |
| Demo plan monthly cap | 10,000 calls | SEARCH-SNIPPET |
| Keyless public API | Lower, variable limits | SEARCH-SNIPPET |
| Endpoint | `GET /api/v3/simple/price?ids=solana&vs_currencies=usd&include_last_updated_at=true` with header `x-cg-demo-api-key` | UNVERIFIED → verify |

At 1 call/60 s, SOL/USD uses ~43,200 calls/month, which exceeds the 10,000 cap. Therefore: **CoinGecko at 1 call per 5 min (8,640/month) as a cross-check only; DexScreener SOL/USDC pair as primary at 30 s.** ([10](10-market-data-spec.md) §9 and the config defaults match this allocation.)

## 8. Anthropic (optional AI)

- SDK: `@anthropic-ai/sdk`. Model IDs are configured, not hard-coded. Defaults ([26](26-configuration-reference.md) `ai.*`): `claude-haiku-4-5-20251001` for latency-sensitive entry review and `claude-sonnet-5` for asynchronous analysis. Verify availability and pricing at implementation (Phase 16) against official Anthropic docs.
- Structured output via tool use (a single tool whose `input_schema` is generated from the zod schema), then zod validation.
- Cost is tracked per call from response usage fields.

## 9. Change log (provider facts)

| Date | Change | By |
|---|---|---|
| 2026-09-27 | Initial provisional table from official-domain search snippets (egress to official docs blocked in the documentation environment). | Docs stage |
| 2026-09-27 | SOL/USD source allocation: DexScreener primary, CoinGecko 5-minute cross-check, because of the Demo monthly cap. [10](10-market-data-spec.md) §9 aligned. | Docs stage (consistency audit) |

## 10. Phase 03 verification checklist

For each provider, Phase 03 must produce `fixtures/providers/<provider>/` recordings and update this doc's `Verification` column:

- [ ] Helius: plan limits, credits per method, per-MB streaming cost applicability, WS connection and subscription limits, `logsSubscribe` mentions limit (Solana allows exactly one address per `mentions` filter; confirm), webhook availability and limits on Free, `getPriorityFeeEstimate` availability.
- [ ] Helius: recorded logs notifications for pump.fun, PumpSwap, Raydium AMM v4, Raydium CPMM trades; recorded account data for each pool type.
- [ ] DexScreener: endpoint paths, batch sizes, field names, rate limits, update cadence (probe), terms.
- [ ] GeckoTerminal: base URL, endpoints, trades coverage, OHLCV limits, rate limit behaviour (headers on 429), freshness, terms.
- [ ] CoinGecko: Demo limits (resolve the 30 vs 100/min conflict), monthly cap, endpoint.
- [ ] Solana constants: program IDs, WSOL/USDC mints, base fee per signature, ATA rent-exempt minimum (`getMinimumBalanceForRentExemption(165)`).
- [ ] Venue fee schedules (pump.fun curve, PumpSwap, Raydium AMM v4, Raydium CPMM) from official program sources/IDLs, recorded in `ref.venues.fee_model` with `verification_ref`.
