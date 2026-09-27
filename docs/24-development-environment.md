# 24 — Development Environment

| Field | Value |
|---|---|
| Status | Draft v0.1 |
| Upstream | [04](04-technical-spec.md), [21](21-security-spec.md), [23](23-testing-strategy.md) |
| Downstream | [DEVELOPMENT.md](../DEVELOPMENT.md), Phase 00 |
| Used by phases | 00, and every phase for commands |

## 1. Prerequisites

| Tool | Version policy | Purpose |
|---|---|---|
| Node.js | Active LTS, pinned in `.nvmrc` (Phase 00 records the exact version) | Runtime |
| pnpm | Pinned via `packageManager` field in root `package.json` (corepack) | Package manager |
| Docker + Docker Compose | Current stable | Local Postgres, Testcontainers, engine image |
| Git | ≥ 2.40 | VCS |
| gitleaks (optional locally, required in CI) | Current stable | Secret scanning |
| chrony/NTP | OS default | Clock discipline (engine host) |

## 2. Pinned versions register

Phase 00 fills this table from the actual lockfile. Later upgrades update it.

| Package | Version | Pinned in phase |
|---|---|---|
| node | TBD | 00 |
| pnpm | TBD | 00 |
| typescript | TBD | 00 |
| vitest | TBD | 00 |
| eslint / typescript-eslint | TBD | 00 |
| prettier | TBD | 00 |
| dependency-cruiser | TBD | 00 |
| zod | TBD | 01 |
| decimal.js | TBD | 01 |
| pino | TBD | 01 |
| pg / kysely / kysely-codegen / node-pg-migrate | TBD | 02 |
| testcontainers | TBD | 02 |
| ws | TBD | 03 |
| @solana/kit | TBD | 03 |
| fastify (+ plugins) | TBD | 18 |
| prom-client | TBD | 04/22 |
| @anthropic-ai/sdk | TBD | 16 |
| next / react / tailwindcss / @tanstack/react-query / lightweight-charts / recharts | TBD | 19 |
| @playwright/test | TBD | 19 |
| fast-check | TBD | 01 |

## 3. Local services (docker-compose.yml)

```yaml
services:
  postgres:
    image: postgres:16            # or 17; pinned digest in Phase 00
    environment:
      POSTGRES_USER: paperbot_owner
      POSTGRES_PASSWORD: localdev   # local only
      POSTGRES_DB: paperbot
    ports: ["5432:5432"]
    volumes: ["pgdata:/var/lib/postgresql/data"]
    healthcheck: { test: ["CMD-SHELL","pg_isready -U paperbot_owner"], interval: 5s, retries: 10 }
volumes: { pgdata: {} }
```

A migration creates the `paperbot_engine` role for local use. Local connection strings live in `.env.local` (gitignored).

## 4. Environment variables

`.env.example` (committed, no values):

```dotenv
# ---- Engine ----
NODE_ENV=development
PAPERBOT_ENV=development            # selects config/env/<env>.yaml
PAPERBOT_STRATEGY_CONFIG=config/strategies/smart-money-wave/0.1.0.yaml
DATABASE_URL=                       # postgres://paperbot_engine:...@localhost:5432/paperbot
DATABASE_OWNER_URL=                 # for migrations only
HELIUS_API_KEY=
HELIUS_WEBHOOK_AUTH=                # optional (webhook wallet source)
COINGECKO_API_KEY=                  # optional
ANTHROPIC_API_KEY=                  # optional (ai.mode=OFF works without)
ENGINE_API_TOKEN=                   # long random string
WS_TOKEN_SECRET=                    # long random string
METRICS_TOKEN=                      # optional
DASHBOARD_ORIGIN=http://localhost:3000
# ---- Dashboard ----
ENGINE_HTTP_URL=http://localhost:8080
NEXT_PUBLIC_ENGINE_WS_URL=ws://localhost:8080/api/v1/stream
NEXT_PUBLIC_APP_ENV=development
DASHBOARD_PASSWORD_HASH=            # argon2id hash (script: pnpm dashboard:hash-password)
SESSION_SECRET=                     # ≥ 32 random bytes
```

**Forbidden:** any variable holding private keys, seed phrases or keypairs. The engine refuses to start ([21](21-security-spec.md) §4).

## 5. Commands

| Command | Does |
|---|---|
| `pnpm install` | Install all workspaces |
| `pnpm verify` | format:check → lint → typecheck → check:safety → test → test:integration (the CI equivalent) |
| `pnpm dev:db` | `docker compose up -d postgres` |
| `pnpm db:migrate` | Apply migrations (owner URL) |
| `pnpm db:codegen` | Regenerate Kysely types from the local DB |
| `pnpm db:partitions` | Ensure partitions ahead / drop expired |
| `pnpm dev:engine` | Run the engine with watch mode (tsx) |
| `pnpm dev:dashboard` | Run Next.js dev server |
| `pnpm replay -- --config <id> --from <iso> --to <iso> [--seed n]` | Replay run |
| `pnpm fixtures:record -- <recorder> [...]` | Record provider fixtures (real network; manual only) |
| `pnpm trace:audit -- --run <id>` | Decision trace completeness |
| `pnpm check:safety` | No-signing/no-secret static scan |

## 6. Editor and hooks

- `.editorconfig`, Prettier, ESLint integration.
- Git hooks (via `simple-git-hooks` or `lefthook`): pre-commit runs `lint-staged` (prettier + eslint on staged files) and `gitleaks protect --staged` if installed. Pre-push runs `pnpm typecheck && pnpm test`.

## 7. CI (GitHub Actions)

`.github/workflows/ci.yml`:
1. Checkout, setup Node (from `.nvmrc`), pnpm via corepack, cache the pnpm store.
2. `pnpm install --frozen-lockfile`
3. `pnpm format:check && pnpm lint && pnpm typecheck`
4. `pnpm check:safety`
5. `pnpm test` (unit + contract + scenario in-memory)
6. `pnpm test:integration` (Testcontainers; Docker available on ubuntu runners)
7. `pnpm build` (engine + dashboard) + bundle secret scan
8. `gitleaks` action
9. (dashboard/api changes) `pnpm test:e2e`

Nightly workflow: mutation tests, full chaos suite, perf smoke.

## 8. Claude Code on the web / remote sessions

For AI coding agents in ephemeral containers: a SessionStart hook (optional, Phase 00) runs `corepack enable && pnpm install --frozen-lockfile` so tests and linters work at session start. Testcontainers requires Docker. If Docker is unavailable in the agent environment, run integration tests in CI and record that in the phase verification notes. Unit, contract and in-memory scenario tests must pass without Docker.
