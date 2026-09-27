# 04 — Technical Specification

| Field | Value |
|---|---|
| Status | Draft v0.1 |
| Upstream | [02](02-product-requirements.md), [03](03-system-requirements.md) |
| Downstream | [05](05-system-architecture.md), [24](24-development-environment.md), every build phase |
| Used by phases | 00, 01, and all later phases for conventions |

## 1. Technology choices (each justified)

| Concern | Choice | Reason | Rejected alternatives |
|---|---|---|---|
| Language | **TypeScript** (strict) everywhere | One language across engine and dashboard. Shared types and zod schemas. Strong ecosystem for Solana RPC and web. | Python engine + TS UI (two type systems, duplicated schemas). Rust (slower iteration for MVP). |
| Python | **Not in MVP runtime.** Optional for offline research notebooks in `research/` (Phase 25+). | Research convenience (pandas) without splitting the runtime. | Python services in the hot path. |
| Runtime | **Node.js active LTS** (pin exact major in `.nvmrc` at Phase 00; expected Node 24.x) | Mature, good WebSocket/HTTP performance for this I/O-bound workload. | Bun/Deno (ecosystem risk for pg, testing). |
| Package manager / monorepo | **pnpm workspaces** | Fast, strict dependency isolation, native workspaces. No extra build orchestrator needed at MVP size. | Turborepo/Nx (add later if build times hurt). |
| Validation | **zod** | Runtime validation + inferred types. Used for provider payloads, config, API contracts, AI outputs. | io-ts, ajv (less ergonomic TS inference). |
| Decimal math | **decimal.js** for prices/ratios. Native **bigint** for raw amounts. | Correct money math ([ADR-0014](32-architecture-decision-records.md#adr-0014-integer-base-units-and-decimal-math)). | Floats (forbidden). |
| HTTP/WS server (engine) | **Fastify** + `@fastify/websocket` | Fast, schema-friendly, first-class TS, plugin model ([ADR-0021](32-architecture-decision-records.md#adr-0021-fastify-for-the-engine-api)). | Express (slower, weaker typing), NestJS (heavy). |
| WS client (providers) | **ws** | De-facto standard, controllable ping/pong and backpressure. | Browser-style polyfills. |
| Solana RPC | **@solana/kit** (web3.js v2 line) wrapped in a read-only allowlisted client | Modern, tree-shakable. Wrapped so signing/sending is unreachable ([21](21-security-spec.md) §5). | web3.js v1 (legacy). Direct raw JSON-RPC is allowed inside the wrapper if kit lacks a method. |
| Database | **PostgreSQL** (16+; Supabase or self-hosted) | Relational integrity for trading records. Native partitioning for time series ([ADR-0003](32-architecture-decision-records.md#adr-0003-postgresql-with-native-partitioning-no-timescaledb-dependency)). | TimescaleDB (availability on Supabase not guaranteed). ClickHouse (extra system). |
| DB access | **Kysely** (typed query builder) + **pg** driver; `kysely-codegen` for types | Type-safe SQL without ORM magic, explicit queries, easy COPY/batch. | Prisma (poor partitioning/COPY story), Drizzle (acceptable; Kysely chosen for raw-SQL fidelity). |
| Migrations | **node-pg-migrate** with plain `.sql` files | SQL-first, reviewable, works with Supabase or any Postgres ([ADR-0020](32-architecture-decision-records.md#adr-0020-kysely-and-sql-first-migrations)). | Supabase CLI migrations only (ties us to Supabase). |
| Logging | **pino** (JSON) | Fast structured logging, redaction support. | winston. |
| Metrics | In-process registry via **prom-client**, exposed at `/metrics` | Standard format. Scrape optional. Latency samples also persisted to DB. | Full OpenTelemetry stack (deferred). |
| Testing | **Vitest** (unit/integration), **Testcontainers** (Postgres), **Playwright** (E2E), **fast-check** (property tests) | Fast TS-native runner. Real Postgres in tests. Property tests for accounting/economics math. | Jest (slower ESM story). |
| HTTP mocking | **undici MockAgent** / **msw** (node) | Deterministic provider contract tests. | Live calls in tests (forbidden). |
| Lint/format | **ESLint** (flat config, typescript-eslint strict-type-checked), **Prettier**, **dependency-cruiser** | Quality + enforced module boundaries. | |
| Dashboard | **Next.js** (App Router, current stable pinned at Phase 19), **React**, **TypeScript**, **Tailwind CSS**, **shadcn/ui** (Radix), **TanStack Query**, **TradingView Lightweight Charts** (price), **Recharts** (stats) | Mature SSR/React stack that deploys to Vercel. Lightweight Charts is purpose-built for financial series (Apache-2.0). | Grafana-only UI (poor product UX). |
| AI | **Anthropic Claude** via official `@anthropic-ai/sdk`, structured output via tool use + zod validation; model IDs from config | Strong structured output. Provider abstracted behind `LlmClient` port. | Hard-coding one model. |
| Containers | **Docker** + Docker Compose (dev: Postgres; prod: engine image) | Reproducible environment and simple deploy. | Kubernetes (not justified). |
| CI | **GitHub Actions** | Repo is on GitHub. | |

**Version policy:** Phase 00 pins exact versions in `package.json` and `pnpm-lock.yaml` and records them in [24](24-development-environment.md) §2. Upgrades are separate commits with passing CI. Do not assume APIs of a library version not pinned.

## 2. Repository layout (canonical)

```text
paperbot/
├─ apps/
│  ├─ engine/                     # Node.js modular monolith (the only long-running backend process)
│  │  ├─ src/
│  │  │  ├─ main.ts               # entrypoint: loads config, runs safety guards, composes modules
│  │  │  ├─ bootstrap/            # composition root, mode guard, env guard, lifecycle
│  │  │  └─ modules/
│  │  │     ├─ pipeline/          # event bus, queues, dedup, event sources, persister, scheduler
│  │  │     ├─ ingestion/         # provider adapters: helius/, dexscreener/, geckoterminal/, coingecko/
│  │  │     ├─ reference/         # tokens, pools, DEX registry, pool selection
│  │  │     ├─ market-state/      # decoders, pool state, market state store, quality, bars, SOL/USD
│  │  │     ├─ wallet-intel/      # wallets, swaps, round trips, scoring
│  │  │     ├─ wave/              # wave features, state machine, scoring, signals
│  │  │     ├─ economics/         # pure cost/impact/latency/failure models
│  │  │     ├─ portfolio/         # ledger, positions, PnL, snapshots, analytics
│  │  │     ├─ execution/         # TradingExecutor, PaperExecutor, Shadow/Real stubs
│  │  │     ├─ risk/              # risk engine
│  │  │     ├─ strategy/          # decision engine, exit manager, run manager
│  │  │     ├─ ai/                # LLM client, agents, schemas, replay cache
│  │  │     ├─ replay/            # replay orchestration
│  │  │     ├─ api/               # Fastify REST + WS
│  │  │     └─ observability/     # metrics, health, latency recorder, provider health
│  │  └─ test/                    # cross-module integration & scenario tests
│  └─ dashboard/                  # Next.js app (Vercel)
├─ packages/
│  ├─ core/                       # pure domain primitives: types, zod schemas, time, ids, money, events. NO I/O.
│  ├─ config/                     # config schema, loader, canonical hashing
│  ├─ db/                         # migrations (SQL), generated Kysely types, connection factory, partition manager
│  └─ api-contract/               # zod schemas for REST/WS payloads shared by engine and dashboard
├─ config/
│  ├─ base.yaml                   # non-strategy engine config (providers, freshness, limits)
│  ├─ env/{development,test,production}.yaml
│  └─ strategies/smart-money-wave/0.1.0.yaml
├─ fixtures/                      # recorded provider payloads & scenario event streams (sanitized)
├─ scripts/                       # safety checks (no-signing scan), partition tooling, fixture recorders
├─ research/                      # (optional, later) notebooks; never imported by runtime
├─ docs/  build/
├─ docker-compose.yml
└─ README.md ARCHITECTURE.md DEVELOPMENT.md CONTRIBUTING.md CLAUDE.md DOCUMENTATION-INDEX.md BUILD-MASTER-PLAN.md
```

Package names: `@paperbot/core`, `@paperbot/config`, `@paperbot/db`, `@paperbot/api-contract`, `@paperbot/engine`, `@paperbot/dashboard`.

## 3. Module anatomy (engine)

Every engine module follows the same shape:

```text
modules/<name>/
├─ index.ts          # public API only (types + factory). Other modules import ONLY from here.
├─ domain/           # pure logic, no I/O (unit-tested, deterministic)
├─ ports.ts          # interfaces this module needs (repositories, clients, clock)
├─ adapters/         # implementations of ports (db repositories, provider clients)
├─ service.ts        # orchestrates domain + ports, subscribes to bus events
└─ __tests__/        # unit tests colocated; integration tests in adapters/__tests__
```

Rules:
- `domain/` imports only `@paperbot/core` and its own module's domain files.
- Modules communicate through **bus events** or through the **public API** in `index.ts`. Never through deep imports.
- Each module owns its database tables ([07](07-database-schema.md) §2). Other modules read through the owner's public query API or DB views. There are no cross-module writes.

## 4. Coding conventions

- **TypeScript**: `strict: true`, `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`, `noImplicitOverride`, `useUnknownInCatchVariables`. ES modules. Target ES2023.
- **Naming**: files `kebab-case.ts`; types `PascalCase`; functions/vars `camelCase`; DB columns `snake_case`; event types `dot.separated.lowercase`; enums as string-literal unions + zod enums (`'ENTER' | 'WAIT' | ...`).
- **Branded types** for identifiers and units: `Lamports`, `RawAmount`, `EpochMicros`, `DurationMicros`, `Bps`, `MintAddress`, `WalletAddress`, `PoolAddress`, `Signature`, `EventId`, `RunId`.
- **Errors**: domain functions return `Result<T, DomainError>` for expected failures. Exceptions only for programmer errors and infrastructure faults. Every error has a stable `code`.
- **No floats for money**. ESLint rule bans `Number(...)`/`parseFloat` on fields typed as amounts. Code review checklist item.
- **No ambient time**. Use the `Clock` port (`clock.now()`, `clock.monotonicMicros()`).
- **No ambient randomness**. Use the `Rng` port seeded per run (`rng.next()`); `Math.random` is banned by lint.
- **Config access** only via the typed config object injected at composition. `process.env` is read only in `bootstrap/`.
- **Logging** via injected logger with bound context (`runId`, `module`, `correlationId`).
- **Comments** explain *why*. TSDoc on public module APIs.

## 5. Core primitives (`@paperbot/core`)

| Primitive | Definition |
|---|---|
| `EpochMicros` | `number` (safe up to year 2255 at µs resolution), branded |
| `Clock` | `{ now(): EpochMicros; monotonicMicros(): number }`. Implementations: `SystemClock`, `SimulatedClock` |
| `Scheduler` | Timer service bound to a `Clock`: `schedule(at: EpochMicros, task)`, `every(interval, task)`. In replay, driven by simulated time. |
| `Rng` | Seeded PRNG (xoshiro128** or similar). `fork(label)` derives an independent stream deterministically. |
| `Id` | UUIDv7 generator (time-ordered). Injected for deterministic tests. |
| `Money` | `Lamports` (bigint), `RawAmount` (bigint + decimals), `UsdDecimal` (Decimal), converters |
| `Result` | `{ ok: true; value } \| { ok: false; error }` |
| `EventEnvelope` | Canonical event shape ([09](09-real-time-data-architecture.md) §3) |
| `DataQuality` | `'FRESH' \| 'STALE' \| 'DEGRADED' \| 'UNKNOWN'` |
| Enums | `RunMode`, `RunSource`, `DecisionAction`, `RiskOutcome`, `OrderStatus`, `OrderSide`, `OrderIntent`, `PositionStatus`, `WaveState`, `WalletStatus`, `PoolType`, `VenueId` |

## 6. Module dependency rules (enforced)

Allowed dependencies (→ means "may import public API of"):

```text
core ← everything
config → core
db → core
api-contract → core

engine modules:
pipeline      → core, config, db
ingestion     → pipeline, core
reference     → pipeline, ingestion (clients), core, db
market-state  → pipeline, reference, core
wallet-intel  → pipeline, reference, market-state (read), ingestion (clients), core, db
wave          → pipeline, market-state (read), wallet-intel (read), reference (read), core
economics     → core                      # pure
portfolio     → pipeline, economics, core, db
execution     → pipeline, economics, market-state (read), portfolio, core, db
risk          → portfolio (read), market-state (read), reference (read), core
ai            → pipeline, core, db        # outputs consumed via bus/ports only
strategy      → pipeline, wave, risk, execution, portfolio (read), ai (port), market-state (read), core, db
replay        → pipeline, strategy, core, db
observability → pipeline, core, db
api           → all modules' public query APIs (read) + strategy/run control commands; api-contract
dashboard     → api-contract, core (types only)
```

Forbidden, as dependency-cruiser errors:
- `execution` importing anything that can sign or send transactions.
- `domain/` folders importing adapters, `pg`, `ws`, `fastify`, `@anthropic-ai/sdk`, or `@solana/*`.
- `dashboard` importing `apps/engine/**` or `@paperbot/db`.
- Any module importing another module's internal path (only `modules/<x>/index.ts`).

## 7. Configuration loading

`@paperbot/config` loads `config/base.yaml`, then the env overlay, then the strategy config file (or a DB config version when starting a run by ID), then env-var overrides for secrets. It validates with zod, freezes, and computes `configHash = sha256(canonicalJson(strategyConfig))`. Full schema: [26](26-configuration-reference.md).

## 8. Error codes

Stable error codes are namespaced (`ING_`, `MKT_`, `EXE_`, `RSK_`, `PFL_`, `AI_`, `CFG_`, `SAF_`). The registry lives in `packages/core/src/errors/codes.ts` and is mirrored in [28](28-decision-logging-spec.md) §6 (reason codes) and [22](22-observability-spec.md) §3.

## 9. Build and run commands (contract)

Defined in Phase 00 and documented in [DEVELOPMENT.md](../DEVELOPMENT.md):

| Command | Purpose |
|---|---|
| `pnpm install` | Install |
| `pnpm lint` / `pnpm format:check` / `pnpm typecheck` | Static checks |
| `pnpm test` | Unit tests (all packages) |
| `pnpm test:integration` | Integration tests (needs Docker for Testcontainers) |
| `pnpm test:e2e` | Playwright E2E (dashboard + engine against fixtures) |
| `pnpm check:safety` | No-signing / no-secret static scan |
| `pnpm db:migrate` / `pnpm db:partitions` | Migrations / partition maintenance |
| `pnpm dev:engine` / `pnpm dev:dashboard` | Local dev |
| `pnpm verify` | Everything CI runs, in order |
