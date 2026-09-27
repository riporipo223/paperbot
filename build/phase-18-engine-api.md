# Phase 18 — Engine API (REST + WebSocket)

| Field | Value |
|---|---|
| Milestone | M5 Operator UI |
| Depends on | 15, 17 |
| Size | L |
| Requirements | FR-UI-002 (server side), FR-LOG-001 (trace endpoint), FR-RSK-004 (pause endpoints), NFR-SEC-003 |

## 1. Objective
Expose every read model and the few control commands through a Fastify REST API and a WebSocket stream. Both are defined by shared zod contracts in `@paperbot/api-contract`, with authentication, CORS, rate limiting, idempotency, and contract tests. Optionally mount the Helius webhook route.

## 2. Context (read first)
- [08](../docs/08-api-spec.md) (entire)
- [21](../docs/21-security-spec.md) §5
- [28](../docs/28-decision-logging-spec.md) §4
- [22](../docs/22-observability-spec.md) §5–6

## 3. Dependencies
Phases 15 and 17 DONE.

## 4. Inputs
All module public query APIs, the RunManager, EntryPause, the trace builder, the comparison service, and the metrics registry.

## 5. Outputs
- `@paperbot/api-contract`: zod schemas for every endpoint and WS message, `API_VERSION`, and a JSON Schema export for the config editor.
- `modules/api`: Fastify server, auth plugins (service token, WS JWT), routes per [08](../docs/08-api-spec.md) §4, the WS hub with channels/snapshots/deltas/seq/backpressure/coalescing, idempotency store, `/health`, `/ready`, `/metrics`, and the webhook route (conditional).
- Contract tests validating every route's responses against the schemas.

## 6. Files To Create
```text
packages/api-contract/src/{common.ts,system.ts,runs.ts,config.ts,portfolio.ts,orders.ts,strategy.ts,market.ts,wallets.ts,analytics.ts,auth.ts,stream.ts,index.ts}
apps/engine/src/modules/api/
  index.ts server.ts
  plugins/{auth-service-token.ts,auth-ws-token.ts,cors.ts,rate-limit.ts,helmet.ts,idempotency.ts,error-handler.ts,api-version.ts}
  routes/{system.ts,runs.ts,control.ts,config.ts,portfolio.ts,positions.ts,orders.ts,waves.ts,signals.ts,decisions.ts,risk.ts,ai.ts,market.ts,wallets.ts,analytics.ts,auth.ts,webhooks.ts,health.ts,metrics.ts}
  ws/{hub.ts,channels.ts,client-session.ts,coalescer.ts}
  __tests__/{contract/*.test.ts,auth.test.ts,ws-hub.test.ts,idempotency.test.ts,cors.test.ts}
```

## 7. Files To Modify
- `bootstrap/composition-root.ts` (start the API last and stop it first)
- `modules/*/index.ts` to add any missing read queries (read-only)

## 8. Database Changes
None (the idempotency store is in memory with a 24 h TTL. A restart loses it; acceptable, since commands are also idempotent at the domain level by key).

## 9. API Changes
This phase implements [08](../docs/08-api-spec.md). Any deviation → update [08](../docs/08-api-spec.md) in the same commit.

## 10. Environment Variables
`ENGINE_API_TOKEN`, `WS_TOKEN_SECRET`, `METRICS_TOKEN` (optional), `DASHBOARD_ORIGIN`, `HELIUS_WEBHOOK_AUTH` (optional).

## 11. Implementation Tasks

**18.1 Contracts package**
- Tests first: schema round-trips for sample payloads. Money fields are strings. Timestamps are ISO with µs. Enums match core.

**18.2 Server skeleton + error handling + health/ready/metrics**
- Tests first (inject): `/health` 200. `/ready` 503 with reasons when the DB is down (fake). `/metrics` requires the token. Errors follow the `ApiError` shape without stack traces.

**18.3 Auth**
- Tests first: missing/invalid bearer → 401. Constant-time comparison (a unit test of the helper). The WS token is minted only with the service token, and has exp ≤ 5 min, the audience, and `jti` replay protection.

**18.4 CORS, rate limit, helmet, API version**
- Tests first: a disallowed origin is rejected. Over the rate → 429. A mismatched major `X-Api-Version` → 426.

**18.5 Read routes (grouped by domain; one commit per group)**
- Tests first (contract): each route returns schema-valid data from a seeded test DB (integration via Testcontainers, populated by running scenario S01 in Postgres mode). Pagination cursors are stable.

**18.6 Decision trace route**
- Tests first: the S01 entry decision returns a complete trace (`completeness.ok = true`), and the narrative lines match the snapshot.

**18.7 Command routes**
- Tests first: `POST /runs` rejects mode ≠ SIMULATION (422 `SAF_MODE_NOT_ALLOWED`). Second LIVE_FEED run → 409. `Idempotency-Key` is required and replays return the cached response. Pause/resume. Manual close (goes through decision→risk→executor; the trace shows `EXIT_MANUAL`). Config validate/store (hash-idempotent). Wallet add/patch/import/backfill.

**18.8 WS hub**
- Tests first (ws client in tests): subscribe → snapshot then deltas with a monotonic seq per channel. Coalescing ≤ 4/s per subject. Client queue overflow → a snapshot resync. Heartbeat ping. An expired token is rejected at the handshake. Unsubscribe stops deltas.

**18.9 Webhook route (conditional)**
- Tests first: only mounted when `wallet_source=helius_webhook`. Auth header validation. The body limit. Enqueue to the adapter and respond 200 quickly.

**18.10 OpenAPI/JSON Schema export (optional but recommended)**
- Generate an OpenAPI document from the zod contracts to `docs/generated/openapi.json` (a CI check keeps it in sync).

## 12. Acceptance Criteria
1. Every endpoint in [08](../docs/08-api-spec.md) is implemented and contract-tested. Deviations are documented.
2. The WS protocol behaves per [08](../docs/08-api-spec.md) §5 (tests).
3. The security controls are tested (auth, CORS, rate limit, idempotency, no stack traces).
4. There is no endpoint capable of real execution or signing (review + safety scan).

## 13. Tests
Contract, integration (routes over a seeded DB), and WS protocol tests.

## 14. Failure Cases
- DB down → read routes 503 with `DB_UNAVAILABLE`. WS keeps the connection and sends a `system` delta.
- Slow consumers → resync path.

## 15. Observability
Request logs (method, route, status, duration; no bodies), `paperbot_http_requests_total`, `paperbot_ws_clients`, and the `ws_broadcast` latency stage.

## 16. Security
Per [21](../docs/21-security-spec.md) §5. Secrets never appear in responses. The config endpoints never return secrets (they are not part of the config objects).

## 17. Verification
```bash
pnpm test --filter @paperbot/api-contract
pnpm test:integration --filter @paperbot/engine -- api
curl -s localhost:8080/health
```

## 18. Commit Strategy
1. `feat(api-contract): add shared rest and ws schemas`
2. `feat(api): add server, error handling, health, ready, metrics`
3. `feat(api): add auth, cors, rate limit, idempotency`
4. `feat(api): add read routes for <group>` (several commits)
5. `feat(api): add decision trace route`
6. `feat(api): add command routes`
7. `feat(api): add websocket hub`
8. `feat(api): add helius webhook route`

## 19. Definition Of Done
- [ ] Acceptance criteria met. Global DoD satisfied.
