# 08 — API Specification (Engine REST + WebSocket)

| Field | Value |
|---|---|
| Status | Draft v0.1 |
| Upstream | [05](05-system-architecture.md), [07](07-database-schema.md) |
| Downstream | [20](20-dashboard-spec.md), [28](28-decision-logging-spec.md), Phase 18 |
| Used by phases | 18 (implementation), 19–21 (consumption), 05 (optional webhook endpoint) |

## 1. Principles

- The engine is the only backend. The dashboard never touches the database.
- All request and response bodies are defined as **zod schemas in `@paperbot/api-contract`**. Engine and dashboard import the same schemas. Changing a schema is a contract change: update this doc and bump `API_VERSION` if it breaks compatibility.
- Base path: `/api/v1`. JSON only. Timestamps are ISO-8601 UTC strings with microseconds. Money and raw amounts are **strings** (to preserve bigint/decimal precision). Enums are uppercase strings.
- Read endpoints are side-effect free. Command endpoints are few, explicit, and idempotent (`Idempotency-Key` header required).
- No endpoint can place a real order, sign, or move funds. There are no such capabilities anywhere in the engine.

## 2. Authentication

| Caller | Mechanism |
|---|---|
| Dashboard server (Next.js route handlers / server components) | `Authorization: Bearer <ENGINE_API_TOKEN>` (long random secret, server-side only) |
| Browser WebSocket | Short-lived token from `POST /api/v1/auth/ws-token` (called by the dashboard server). HMAC-SHA256 JWT, `exp` ≤ 5 min, `aud = "paperbot-ws"`, `scope = "stream:read"`. Passed as the `?token=` query param on connect (only for the handshake; re-auth before expiry by reconnecting). |
| Helius webhook (optional) | `Authorization` header value configured in Helius, compared in constant time with `HELIUS_WEBHOOK_AUTH`. |
| `/health`, `/ready` | Unauthenticated (no sensitive data). |
| `/metrics` | Bearer `METRICS_TOKEN`, or bound to a private interface. |

CORS: allow only `DASHBOARD_ORIGIN`. Rate limit: 60 req/s per token (configurable). Request bodies are capped at 256 KB (webhooks at 1 MB).

## 3. Common shapes

```ts
// Pagination (cursor-based, stable on UUIDv7/time ordering)
type Page<T> = { items: T[]; nextCursor: string | null };
// Query: ?limit=50&cursor=<opaque>

// Error
type ApiError = { error: { code: string; message: string; details?: unknown; requestId: string } };
// HTTP: 400 validation, 401 auth, 403 forbidden, 404 not found, 409 conflict/idempotency, 422 domain rule, 429 rate limit, 503 degraded/unavailable

// Quality-annotated value
type Q<T> = { value: T | null; quality: 'FRESH'|'STALE'|'DEGRADED'|'UNKNOWN'; ageMs: number | null; source: string | null };
```

## 4. REST endpoints

### 4.1 System

| Method | Path | Description |
|---|---|---|
| GET | `/health` | Liveness: `{status:'ok', version, uptimeS}` |
| GET | `/ready` | Readiness: DB reachable, config loaded, streams initialized. 503 with reasons otherwise. |
| GET | `/api/v1/system/status` | Engine mode, active run, entries paused flag + reasons, degraded components, clock offset |
| GET | `/api/v1/system/providers` | Per provider/channel: status, last message age, reconnect count, rate-limit hits, credit usage (hour/day/month vs budget) |
| GET | `/api/v1/system/streams` | Active subscriptions (wallets, pools), per-stream quality and last event age |
| GET | `/api/v1/system/events?level&type&from&to` | Paged `ops.system_events` |
| GET | `/api/v1/system/gaps?resolution` | Paged `ops.data_gaps` |
| GET | `/metrics` | Prometheus text format |

### 4.2 Runs and config

| Method | Path | Description |
|---|---|---|
| GET | `/api/v1/runs?status&source` | Paged runs |
| GET | `/api/v1/runs/:runId` | Run detail (versions, seed, sources, status, timing) |
| POST | `/api/v1/runs` | Start a run: `{configVersionId, source:'LIVE_FEED'} \| {configVersionId, source:'REPLAY', replayFrom, replayTo, replayOrdering, rngSeed?, replayOfRunId?}`. `mode` is implied `SIMULATION`. Any `mode` other than `SIMULATION` returns 422 `SAF_MODE_NOT_ALLOWED`. Only one `LIVE_FEED` run may be `RUNNING` at a time (409). |
| POST | `/api/v1/runs/:runId/stop` | Stop a run gracefully |
| POST | `/api/v1/control/entries/pause` | `{reason}`: pause new entries (exits continue) |
| POST | `/api/v1/control/entries/resume` | Resume entries (rejected if automatic pause reasons are still active) |
| GET | `/api/v1/strategies` | Strategies + versions |
| GET | `/api/v1/config/versions?strategyVersionId` | Config versions |
| GET | `/api/v1/config/versions/:id` | Full config JSON + hash |
| POST | `/api/v1/config/validate` | Validate a config body. Returns field errors. |
| POST | `/api/v1/config/versions` | Store a new immutable config version. Returns the existing one if the hash matches (idempotent). |
| GET | `/api/v1/config/schema` | JSON Schema (generated from zod) for editor UIs |

### 4.3 Portfolio, positions, orders

| Method | Path | Description |
|---|---|---|
| GET | `/api/v1/runs/:runId/portfolio` | Current: cash (raw + USD), rent locked, equity (USD/SOL), exposure, realized/unrealized PnL, ROI, fees, slippage cost, drawdown, open positions count, mark quality |
| GET | `/api/v1/runs/:runId/portfolio/snapshots?from&to&maxPoints` | Equity curve series (server-side downsampling to `maxPoints`) |
| GET | `/api/v1/runs/:runId/positions?status` | Positions with live marks (`Q<price>`), PnL, holding time, fees, exit plan state |
| GET | `/api/v1/positions/:positionId` | Position detail with orders, fills, transitions |
| POST | `/api/v1/positions/:positionId/close` | Manual paper exit (`intent=MANUAL`). Goes through decision → risk → PaperExecutor. Idempotent. |
| GET | `/api/v1/runs/:runId/orders?status&side` | Orders |
| GET | `/api/v1/orders/:orderId` | Order + fills + transitions + economics breakdown |
| GET | `/api/v1/runs/:runId/trades` | Closed positions as trades (entry/exit, net PnL, costs, hold time) |

### 4.4 Strategy intelligence

| Method | Path | Description |
|---|---|---|
| GET | `/api/v1/runs/:runId/waves?state&activeOnly` | Waves with current score and features |
| GET | `/api/v1/waves/:waveId` | Wave detail + transitions + feature timeline |
| GET | `/api/v1/runs/:runId/signals?kind` | Signals |
| GET | `/api/v1/signals/:signalId` | Signal detail (features, breakdown, quality) |
| GET | `/api/v1/runs/:runId/decisions?action&reason` | Decisions |
| GET | `/api/v1/decisions/:decisionId/trace` | **Decision trace** ([28](28-decision-logging-spec.md) §4): the full causal chain |
| GET | `/api/v1/runs/:runId/risk-decisions?outcome` | Risk verdicts with checks |
| GET | `/api/v1/ai/outputs?agent&subjectKind&subjectId&runId` | AI outputs (validated output + status + model) |
| GET | `/api/v1/ai/status` | Per-agent: enabled, mode, last call, error rate, budget used |

### 4.5 Market

| Method | Path | Description |
|---|---|---|
| GET | `/api/v1/market/watchlist` | Watched tokens/pools with `Q<>` price, liquidity, volume m5, buys/sells m5, wave state |
| GET | `/api/v1/market/tokens/:mint` | Token reference + latest safety observation + pools + market state |
| GET | `/api/v1/market/pools/:address/bars?interval&from&to` | OHLCV bars (derived; falls back to GeckoTerminal bars where flagged) |
| GET | `/api/v1/market/pools/:address/trades?from&limit` | Recent trades (tracked-wallet trades flagged) |
| GET | `/api/v1/market/reference-prices/SOL-USD` | Current SOL/USD `Q<>` |

### 4.6 Wallets

| Method | Path | Description |
|---|---|---|
| GET | `/api/v1/wallets?status&minScore&sort` | Wallets with current score, confidence, flags, last activity |
| GET | `/api/v1/wallets/:address` | Detail: score components, history stats, AI label (advisory) |
| GET | `/api/v1/wallets/:address/swaps?from&to` | Swaps (live + backfill) |
| GET | `/api/v1/wallets/:address/round-trips` | Reconstructed round trips |
| POST | `/api/v1/wallets` | Add `{address, label?, status?, tags?}`. Validates base58/length. Idempotent on address. |
| PATCH | `/api/v1/wallets/:address` | Update `status`, `label`, `tags`, `notes` |
| POST | `/api/v1/wallets/import` | Bulk import (JSON array or CSV text; ≤ 1,000 rows). Returns per-row results. |
| POST | `/api/v1/wallets/:address/backfill` | Queue a history backfill (credit-budgeted) |

### 4.7 Analytics

| Method | Path | Description |
|---|---|---|
| GET | `/api/v1/runs/:runId/analytics/performance` | All metrics in [18](18-portfolio-accounting-spec.md) §7 |
| GET | `/api/v1/analytics/compare?runIds=a,b,c` | Side-by-side metrics |
| GET | `/api/v1/analytics/latency?stage&from&to&runId` | p50/p90/p95/p99/max + histogram per stage |
| GET | `/api/v1/runs/:runId/analytics/costs` | Cost decomposition (LP fees, network, priority, rent, slippage, impact, adverse move) |

### 4.8 Auth and webhooks

| Method | Path | Description |
|---|---|---|
| POST | `/api/v1/auth/ws-token` | Service-token only. Returns `{token, expiresAt}`. |
| POST | `/api/v1/webhooks/helius` | Only if `ingestion.wallet_source = helius_webhook`. Validates the auth header and payload schema, then hands off to the pipeline with `received_at` stamped at request start. Returns 200 fast (after enqueue). |

## 5. WebSocket stream

Endpoint: `wss://<engine>/api/v1/stream?token=<ws-token>`

### 5.1 Protocol

```ts
// client → server
{ op: 'subscribe',   channels: Channel[] }
{ op: 'unsubscribe', channels: Channel[] }
{ op: 'ping', t: number }

// server → client
{ op: 'snapshot', channel, seq, at, data }      // sent on subscribe
{ op: 'delta',    channel, seq, at, type, data } // incremental update
{ op: 'pong', t }
{ op: 'error', code, message }
```

- `seq` is monotonic **per channel**. If a client sees a gap it must re-subscribe to the channel, which triggers a fresh `snapshot`.
- Server heartbeat: WS ping every 15 s. The client reconnects with backoff if no traffic arrives for 45 s.
- Backpressure: the server keeps a bounded send queue per client (`api.ws.max_queue`, default 1,000). On overflow it drops that client's queued deltas for the channel and sends a new snapshot (latest-wins). Market ticks are coalesced to at most 4/s per subject.

### 5.2 Channels

| Channel | Snapshot | Delta types |
|---|---|---|
| `portfolio:<runId>` | portfolio summary | `portfolio.updated` |
| `positions:<runId>` | open positions | `position.opened`, `position.updated`, `position.closed` |
| `orders:<runId>` | recent orders | `order.created`, `order.submitted`, `order.filled`, `order.failed` |
| `signals:<runId>` | recent signals + active waves | `wave.updated`, `signal.emitted` |
| `decisions:<runId>` | recent decisions | `decision.made`, `risk.evaluated` |
| `market:watchlist` | watchlist | `market.state.changed` (coalesced) |
| `market:pool:<address>` | pool state + open bar | `pool.tick`, `bar.updated`, `trade.observed` |
| `wallets:activity` | recent tracked swaps | `wallet.swap.detected`, `wallet.score.updated` |
| `ai:<runId>` | recent AI outputs | `ai.output.recorded` |
| `system` | status + providers | `provider.status.changed`, `entries.paused`, `entries.resumed`, `system.event`, `latency.summary` (every 10 s) |

## 6. Versioning and compatibility

- `API_VERSION` constant in `@paperbot/api-contract`. The dashboard sends `X-Api-Version`. The engine rejects a mismatched major version with 426.
- Additive changes (new optional fields, new channels) are non-breaking.
- Contract tests (Phase 18) validate every engine response against the shared zod schemas.
