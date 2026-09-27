# 10 — Market Data Specification

| Field | Value |
|---|---|
| Status | Draft v0.1 |
| Upstream | [09](09-real-time-data-architecture.md), [27](27-data-provider-reference.md), [06](06-data-architecture.md) |
| Downstream | [11](11-wallet-intelligence-spec.md), [12](12-wave-detection-spec.md), [16](16-paper-trading-spec.md), [17](17-economics-engine-spec.md), [20](20-dashboard-spec.md) |
| Used by phases | 05, 06, 07, 08 |

## 1. Principle: one internal market state

No consumer fetches its own prices. Every consumer (strategy, executor, portfolio marks, AI context, dashboard, analytics) reads `MarketStateView` from the `market-state` module, and every value comes with `{quality, ageMs, source}`. Third-party chart UIs are never a source of truth.

## 2. Data tiers

| Tier | Purpose | Source | Class | Used for fills? |
|---|---|---|---|---|
| **T1 On-chain pool state** | Reserves → mid price, liquidity, price impact | Helius WS `accountSubscribe` on the pool state account (and vaults where needed); `getAccountInfo` on (re)subscribe | RT-STREAM | **Yes (only source)** |
| **T1 On-chain trades** | Trade-level flow: buyers, sells, sizes, unique traders | Helius WS `logsSubscribe({mentions:[pool or mint]})` + per-venue log decoders | RT-STREAM | No (features only) |
| **T1 Wallet swaps** | Tracked-wallet buys/sells | Helius WS `logsSubscribe({mentions:[wallet]})` + decoder or `getTransaction` enrichment; or Helius webhook ([ADR-0009](32-architecture-decision-records.md#adr-0009-wallet-activity-source)) | RT-STREAM | No |
| **T2 Market snapshots** | Aggregates: volume m5/h1, buy/sell counts, liquidity, FDV, pair age | DexScreener pairs endpoint (primary), GeckoTerminal pool endpoint (fallback) | NRT-POLL | No |
| **T2 Trade reference** | Backfill after gaps, cross-check, unique buyer fallback | GeckoTerminal pool trades (latest 300 within 24 h) | NRT-POLL | No |
| **T2 SOL/USD** | USD conversion | CoinGecko simple price (primary if key configured) / DexScreener SOL-USDC pair / GeckoTerminal | NRT-POLL | Conversion only |
| **T3 Reference/history** | OHLCV history, pool discovery, token lists, wallet history | GeckoTerminal OHLCV, DexScreener search/token endpoints, Helius RPC history | REFERENCE | No |

## 3. Watch-set policy

Subscriptions are scarce (connection, subscription and credit limits). `strategy` owns the **WatchSetPolicy**, and `ingestion` executes it.

| Priority | Subject | Streams | Released when |
|---|---|---|---|
| P0 | Pools of **open positions** | T1 state + T1 trades + T2 snapshots | Position closed + `watch.linger_seconds` (120) |
| P1 | Pools of **active waves** (SEEDED/BUILDING/CONFIRMED) | T1 state + T1 trades + T2 snapshots | Wave terminal state + linger |
| P2 | **Tracked wallets** (status `TRACKED`) | T1 wallet swaps | Status change |
| P3 | SOL/USD | T2 | Never |

- Capacity limits come from config `ingestion.helius.max_ws_connections`, `max_subscriptions_per_connection`, and `watch.max_watched_pools` (default 20). If they are exceeded, the lowest-priority P1 waves (lowest score) are released first. If P0 cannot be served, raise `WATCH_CAPACITY_CRITICAL` (ERROR) and pause entries.
- Tracked wallets beyond WS capacity: switch to the webhook source (if configured) or reduce tracking to the top-N wallets by score (`watch.max_tracked_wallets`). This is logged.

## 4. Supported venues and decoders (MVP)

Only constant-product style venues are supported, because the economics engine can price them exactly from reserves ([ADR-0010](32-architecture-decision-records.md#adr-0010-cp-only-pool-economics-in-mvp)). Program IDs, account layouts, event layouts and fees **MUST be verified against official IDLs or program sources in Phase 03/06/07** before `ref.venues.supported` is set to true. The values below are the working hypothesis.

| Venue id | Program ID (to verify) | Pool type | State account → reserves | Trade log event (to verify) |
|---|---|---|---|---|
| `pumpfun_curve` | `6EF8rrecthR5Dkzon8Nwu78hRvfCKubJ14M5uBEwF6P` | `CP_VIRTUAL_CURVE` | Bonding curve account: virtual token reserves, virtual SOL reserves, real reserves, `complete` flag | Anchor `TradeEvent` in `Program data:` logs (mint, sol amount, token amount, is_buy, user, timestamp, virtual reserves) |
| `pumpswap` | `pAMMBay6oceH9fJKBRHGP5D4bD4sWpmSwMn52FMfXEA` | `CP_AMM` | Pool account → base/quote vault token accounts (vault balances are the reserves) | Anchor `BuyEvent`/`SellEvent` |
| `raydium_amm_v4` | `675kPX9MHTjS2zt1qfr1NYHuzeLXfQM9H24wFSUt1Mp8` | `CP_AMM` | AMM state + vault token accounts (reserves = vault balances adjusted by pending PnL fields, per official source) | `ray_log` (base64) with amounts. No trader field → trader from `getTransaction` only when needed. |
| `raydium_cpmm` | `CPMMoo8L3F4NbTegBCKVNunggL7H1ZpdTHKxQB5qKP1C` | `CP_AMM` | Pool state + vaults (minus accrued protocol/fund fees, per official source) | Anchor swap event (to verify) |

Unsupported, and **rejected** with `UNSUPPORTED_POOL_TYPE`: Orca Whirlpool (CLMM), Raydium CLMM, Meteora DLMM, orderbooks, and any unknown program.

Well-known mints: WSOL `So11111111111111111111111111111111111111112`, USDC `EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v` (verify in Phase 03).

### 4.1 Decoder contract

```ts
interface VenueDecoder {
  venueId: VenueId;
  programId: string;
  // Parse the pool state account data → reserves. Pure; unit-tested against recorded account data.
  decodePoolState(accountData: Uint8Array, ctx: { slot: number; pool: PoolRef }): Result<PoolReserves, DecodeError>;
  // Parse a logs notification → zero or more trades. Pure; unit-tested against recorded logs.
  decodeTradeLogs(logs: string[], ctx: { signature: string; slot: number; pool?: PoolRef }): Result<DecodedTrade[], DecodeError>;
  // Optional: parse a full transaction (getTransaction jsonParsed) → swaps per owner.
  decodeTransaction?(tx: ParsedTransaction, ctx): Result<DecodedSwap[], DecodeError>;
}
```

Decoders are **pure** and validated with recorded fixtures (`fixtures/venues/<venue>/...`), captured in Phase 03/07 from real mainnet data. Any decode error is counted, and the source event is dead-lettered. It is never guessed.

### 4.2 Generic balance-delta swap parser (fallback for wallet swaps)

For a tracked wallet's transaction obtained via `getTransaction` (`jsonParsed`, `maxSupportedTransactionVersion: 0`):

1. Compute the owner-level deltas from `preTokenBalances`/`postTokenBalances` filtered by `owner == wallet`, plus the SOL delta from `preBalances`/`postBalances` for the wallet account index, adjusted by the fee if the wallet is the fee payer. Treat WSOL token deltas as SOL.
2. Exactly one non-SOL mint with a positive delta and negative SOL → `BUY`. Negative token delta and positive SOL → `SELL`. Anything else (multi-hop, token-token, transfers) → `UNSUPPORTED_SWAP_SHAPE`, recorded but not used for waves.
3. Venue attribution: the first invoked program (outer or inner instruction) matching a known venue program ID. The pool is the venue-specific account at the known index (per decoder).
4. `parseMethod = 'BALANCE_DELTA'` is recorded, versus `'VENUE_EVENT'` from log decoders.

## 5. Price and liquidity computation

For CP pools with base reserve `B` (raw, decimals `dB`) and quote reserve `Q` (raw, decimals `dQ`, usually SOL 9):

- Mid price (quote per whole base): `P = (Q / 10^dQ) / (B / 10^dB)`, computed with decimal.js at ≥ 40 significant digits.
- For `CP_VIRTUAL_CURVE`, use the **virtual** reserves for price and impact. Use the real reserves for "can this sell be filled" limits.
- `price_usd = P × SOL_USD` (when quote is SOL) or `P` (when quote is USDC).
- Liquidity (USD) = `2 × (Q / 10^dQ) × SOL_USD` for CP AMMs (the standard convention). For virtual curves, report `real_quote_reserve × SOL_USD` as liquidity, and flag it `liquidity_basis = REAL_QUOTE`.
- All derived values inherit the worst quality of their inputs (reserves, SOL/USD).

## 6. Market state model (in memory)

```ts
interface PoolMarketState {
  pool: PoolRef;                       // address, venue, base/quote mints+decimals
  reserves: Q<PoolReserves & { slot: number; eventId: EventId }>;
  midPriceQuote: Q<Decimal>;
  midPriceUsd: Q<Decimal>;
  liquidityUsd: Q<Decimal>;
  curveComplete?: boolean;             // pump curve → migrated
  trades: TradeRingBuffer;             // last N minutes of decoded trades (dedup by sig+logIndex)
  tradeStreamHealth: 'HEALTHY' | 'STALE' | 'GAP';
  snapshot: Q<MarketSnapshot>;         // latest T2 snapshot
  bars: { '1m': BarBuilder; '5m': BarBuilder; '1h': BarBuilder };
  lastUpdatedAt: EpochMicros;
}
interface TokenMarketState {
  mint: string;
  primaryPool?: PoolAddress;           // selected by reference module
  pools: PoolAddress[];
  safety: Q<TokenSafety>;
}
interface MarketStateView {
  getPool(address): PoolMarketState | undefined;
  getToken(mint): TokenMarketState | undefined;
  getSolUsd(): Q<Decimal>;
  quality(address, consumer: FreshnessConsumer): QualityVerdict; // applies per-consumer max ages (09 §7)
  subscribe(listener): Unsubscribe;    // market.state.changed
}
```

Quality is **computed at read time** against the consumer's max age, so a value silently aging past its threshold is never reported as `FRESH`.

## 7. Primary pool selection

Given all known pools of a token (from DexScreener/GeckoTerminal discovery + on-chain):
1. Filter to supported venues with status `ACTIVE`, where the quote is WSOL (MVP; USDC-quoted pools are supported for pricing, but entries require a SOL quote since cash is SOL).
2. For `pumpfun_curve` with `complete = true` → the curve is migrated. Use the migrated pool (PumpSwap/Raydium) and record `migrated_to`.
3. Choose the highest-liquidity candidate. Break ties by the most recent trade.
4. If no supported pool exists but unsupported pools do → token `REJECTED: UNSUPPORTED_POOL_TYPE`.
5. Re-evaluate on `pool.discovered` or curve completion. Switching the primary pool of an **open position** is an explicit event (`pool.selected{reason:'MIGRATION'}`). Exits then route to the new pool.

## 8. OHLCV

- **Derived bars** (source `DERIVED`) are built from decoded on-chain trades for watched pools: open = first trade price in the bucket, close = last, high/low, volumes, buy/sell counts. With no trades in a bucket, carry forward the close (no volume) only if the reserves were observed in that bucket. Otherwise leave a gap.
- **Reference bars** (source `GECKOTERMINAL`) are fetched on demand for chart history before the watch began (1m/5m/1h, max `limit` per call per [27](27-data-provider-reference.md)). They are cached in `market.ohlcv_bars` and never mixed into strategy features.
- Charts show derived bars where they exist and reference bars elsewhere, with a visual distinction.

## 9. SOL/USD reference

- Primary: DexScreener SOL/USDC top pair (by liquidity) every `ingestion.sol_usd.poll_ms` (default 30,000). Secondary: GeckoTerminal SOL/USDC pool. Cross-check: CoinGecko `simple/price` every 5 minutes when a Demo key is configured. The Demo plan's monthly call cap rules out using it as the primary source ([27](27-data-provider-reference.md) §7).
- Cross-check: if two sources differ by > `market_state.sol_usd_max_divergence_bps` (default 100), quality becomes `DEGRADED` and a WARN is raised.
- The SOL/USD value used for every fill and snapshot is stored with it (`sol_usd` columns).

## 10. Token safety facts (from RPC)

Fetched on `token.discovered` and refreshed every `reference.safety_refresh_minutes` (10) while watched:
- Mint account (`getAccountInfo` jsonParsed): `mintAuthority`, `freezeAuthority`, `supply`, `decimals`, token program, Token-2022 extensions (reject `transferFeeConfig`, `permanentDelegate`, `nonTransferable`, `transferHook`, `defaultAccountState=frozen` per risk config).
- Optional: `getTokenLargestAccounts` → top-10 holder share (excluding known pool/curve vaults).

Stored in `ref.token_safety_observations`, and consumed by the risk token gates ([15](15-risk-engine-spec.md) §4.3).

## 11. Market data acceptance tests (summary)

- Decoders reproduce reserves and trades from recorded fixtures exactly.
- Price computed from reserves matches a hand-calculated expected value to 1e-12 relative error.
- Out-of-order slot updates are ignored.
- Quality transitions `FRESH → STALE` happen exactly when age crosses the consumer threshold (simulated clock tests).
- Primary pool selection handles migration.
- Cross-source trade dedup works (on-chain + GeckoTerminal same signature → one trade).
