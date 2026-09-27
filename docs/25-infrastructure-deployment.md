# 25 — Infrastructure and Deployment

| Field | Value |
|---|---|
| Status | Draft v0.1 |
| Upstream | [05](05-system-architecture.md) §10, [06](06-data-architecture.md) §4, [21](21-security-spec.md), [22](22-observability-spec.md) |
| Downstream | Phase 24 (deployment), Phase 25 (validation run) |
| Used by phases | 24, 25 |

## 1. Environments

| Env | Engine | DB | Dashboard | Data |
|---|---|---|---|---|
| `development` | Local (`pnpm dev:engine`) | Local Docker Postgres | Local Next.js | Real providers (dev keys) or fixture replay |
| `test` | In-process (tests) | Testcontainers | — | Fixtures only |
| `production` (paper) | Docker container on a persistent host | Hosted Postgres, same region | Vercel | Real providers |

"Production" still means **SIMULATION**. It is the long-running paper environment.

## 2. Engine hosting ([ADR-0005](32-architecture-decision-records.md#adr-0005-engine-hosted-on-a-persistent-container-host-not-vercel))

Requirements: always-on process, outbound WSS, inbound HTTPS (API; webhook if used), ~0.5–1 vCPU, 512 MB–1 GB RAM, persistent logs, automatic restart, a region close to the Helius endpoint and the DB.

| Option | Pros | Cons | Est. cost |
|---|---|---|---|
| **Fly.io** machine | Simple Docker deploy, regions, health checks, auto-restart, TLS | Pricing changes over time (verify) | ~$5–10/mo class |
| Railway / Render | Very simple | Sleep policies on low tiers (must be disabled) | ~$5–10/mo |
| Hetzner/DigitalOcean VPS + Docker Compose + Caddy | Cheapest, full control, can co-host Postgres | You operate the OS, updates and TLS | ~$5/mo |

**Recommendation (decision point, [ADR-0018](32-architecture-decision-records.md#adr-0018-database-hosting))**: a single small VPS running engine + Postgres via Docker Compose + Caddy (TLS), with nightly `pg_dump` to object storage. It gives the lowest latency (DB on localhost), enough disk for the event log, and the lowest cost. Alternative: Fly.io engine + Supabase Pro. The operator chooses in Phase 24. Both are documented.

## 3. Database hosting options

| Option | Fit | Notes |
|---|---|---|
| Supabase Free | Dev/demo only | Storage quota far below the ~6 GB/30-day event-log estimate; free projects may pause when inactive (verify current policy) |
| Supabase Pro | OK | Larger storage (verify quota/pricing), backups, managed. Network latency engine↔DB if not co-located (choose the same region). |
| Self-hosted Postgres (Docker on the engine VPS) | **Recommended for paper production** | Local latency, large cheap disk. You own backups (nightly `pg_dump` + WAL optional). |

The schema is identical in all cases. Only the connection string changes.

## 4. Container image

`apps/engine/Dockerfile` (multi-stage):
1. `node:<lts>-slim` builder: `corepack enable`, `pnpm install --frozen-lockfile`, `pnpm --filter @paperbot/engine... build` (tsc) → `pnpm deploy --prod` into `/out`.
2. Runtime `node:<lts>-slim`: non-root user, `COPY /out`, `ENV NODE_ENV=production`, `HEALTHCHECK` hitting `/health`, `CMD ["node","dist/main.js"]`.
- No secrets baked into the image. No `.env` copied. Labels: git SHA, build date.
- The image runs `check:safety` at build time (fails the build if banned patterns appear).

## 5. Production topology (recommended VPS variant)

```text
VPS (Ubuntu LTS, chrony enabled, unattended-upgrades, ufw: 22/80/443 only)
└─ docker compose
   ├─ caddy      (TLS for engine.example.com → engine:8080; auto HTTPS)
   ├─ engine     (paperbot engine image; restart: unless-stopped; env from /etc/paperbot/engine.env, chmod 600)
   ├─ postgres   (volume on local disk; not exposed publicly)
   └─ backup     (cron container: nightly pg_dump → object storage, 14-day retention)
Vercel
└─ dashboard (env: ENGINE_HTTP_URL, ENGINE_API_TOKEN, NEXT_PUBLIC_ENGINE_WS_URL, DASHBOARD_PASSWORD_HASH, SESSION_SECRET)
```

## 6. Deployment procedure (runbook)

1. CI green on `main` (or the release branch).
2. Build and push the engine image (GitHub Actions → GHCR) tagged with the git SHA.
3. On the host: `docker compose pull engine && docker compose run --rm engine node dist/migrate.js` (migrations with the owner URL), then `docker compose up -d engine`.
4. Verify: `/ready` 200, System page streams connected, the startup banner shows `MODE=SIMULATION`.
5. Dashboard: Vercel deploys from `main` automatically (env configured in the Vercel project).
6. Rollback: `docker compose up -d engine` with the previous tag. Migrations are forward-only and must be backward compatible for one version (expand/contract pattern).

Graceful shutdown: SIGTERM → stop accepting new signals → finish in-flight handlers → flush the persister → snapshot the portfolio → close sockets (≤ 20 s). Open positions remain `OPEN` and resume on start.

## 7. Operations

- **Logs**: `docker logs` / Fly logs. Optional log drain (e.g. Better Stack/Grafana Cloud free tiers; verify terms).
- **Metrics**: optional Grafana Cloud free tier scraping `/metrics` with `METRICS_TOKEN`. The dashboard System page is the primary ops UI.
- **Backups**: nightly `pg_dump -Fc` to object storage. Monthly restore test (runbook step). Event-log archives (NDJSON.gz) per retention policy.
- **Partition maintenance**: engine scheduler daily + `pnpm db:partitions` manual.
- **Secrets rotation**: rotate `ENGINE_API_TOKEN`, `WS_TOKEN_SECRET` and provider keys quarterly or on suspicion. The procedure is to update the env and restart.
- **Uptime**: container restart policy. Optional external uptime check on `/health`.

## 8. Cost envelope (MVP paper production, estimates to verify)

| Item | Est. monthly |
|---|---|
| VPS (engine + Postgres) | ~$5–10 |
| Helius | $0 (Free) → $49 (Developer) if validation shows the need |
| DexScreener / GeckoTerminal | $0 |
| CoinGecko | $0 (Demo, optional) |
| Anthropic | ≤ `ai.daily_budget_usd × 30` (default ≤ $30; $0 with `ai.mode=OFF`) |
| Vercel | $0 (Hobby; verify that the Hobby terms permit this personal use) |
| Object storage for backups | ~$0–1 |
