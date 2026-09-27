# Phase 02 — Database Foundation

| Field | Value |
|---|---|
| Milestone | M0 Foundation |
| Depends on | 01 |
| Size | M |
| Requirements | FR-SAF-002 (DB layer), FR-CFG-002/003 (tables), NFR-DATA-002/003 |

## 1. Objective
Stand up the PostgreSQL foundation: migration tooling, the first four migrations (schemas, ops, strategy core, event log), the partition manager, Kysely types, a connection factory, and Testcontainers-based integration testing. Register the strategy and config versions in the DB.

## 2. Context (read first)
- [07](../docs/07-database-schema.md) §1, §3.2 (`market.events`), §3.4 (strategies/versions/config/runs), §3.6, §5–9
- [06](../docs/06-data-architecture.md) §3–7
- ADR-0003, ADR-0017, ADR-0020

## 3. Dependencies
Phase 01 DONE.

## 4. Inputs
Core primitives, config package, Docker Compose Postgres.

## 5. Outputs
- `packages/db`: migrations 0001–0004, a migration runner script, a connection factory (pool with statement timeout), the partition manager (`ensurePartitions`, `dropExpiredPartitions`), generated Kysely types, and test helpers (`withTestDatabase()` via Testcontainers).
- Engine: a `strategy-registry` step at bootstrap that upserts `strategy.strategies`, `strategy_versions` (semver + git SHA) and `config_versions` (hash-idempotent).
- CLI scripts: `db:migrate`, `db:codegen`, `db:partitions`.

## 6. Files To Create
```text
packages/db/migrations/0001_create_schemas.sql
packages/db/migrations/0002_ops_tables.sql
packages/db/migrations/0003_strategy_core.sql
packages/db/migrations/0004_market_events.sql
packages/db/src/{connection.ts,migrate.ts,partitions.ts,types.generated.ts,index.ts}
packages/db/src/test-utils/{testcontainers.ts,index.ts}
packages/db/test/{migrations.test.ts,constraints.test.ts,partitions.test.ts}
apps/engine/src/bootstrap/strategy-registry.ts
apps/engine/src/bootstrap/__tests__/strategy-registry.int.test.ts
```

## 7. Files To Modify
- Root `package.json` scripts (`db:*`, `test:integration`)
- `.github/workflows/ci.yml` (integration job)
- `apps/engine/src/main.ts` (connect DB, run the registry step)
- [24](../docs/24-development-environment.md) §2 versions

## 8. Database Changes
Migrations 0001–0004 exactly as specified in [07](../docs/07-database-schema.md) §3, restricted to:
- 0001: `create schema` for ref, market, intel, strategy, sim, ops. Create roles `paperbot_engine` (DML only) and grants (guarded `DO $$ ... $$` so it is idempotent on Supabase).
- 0002: all `ops.*` tables (partitioned `latency_samples`, `provider_health`).
- 0003: `strategy.strategies`, `strategy_versions`, `config_versions`, `runs` (with the `mode` CHECK excluding LIVE).
- 0004: `market.events` (partitioned) + indexes + helper SQL function `ops.ensure_daily_partition(table regclass, day date)`.
- RLS enabled (deny-all) on all tables.

## 9. API Changes
None.

## 10. Environment Variables
`DATABASE_URL`, `DATABASE_OWNER_URL` (migrations).

## 11. Implementation Tasks

**02.1 Test harness**
- Test first: `withTestDatabase()` starts Postgres (Testcontainers), applies migrations, and yields a Kysely instance. A trivial test queries `select 1`.
- Implement: helper with container reuse per test file and a schema reset between tests.

**02.2 Migration runner**
- Test first: migrations apply on an empty DB. Re-running is a no-op. The migration table records versions.
- Implement: `node-pg-migrate` programmatic runner using SQL files (owner URL).

**02.3 Migrations 0001–0003**
- Tests first (`constraints.test.ts`): inserting a run with `mode='LIVE'` fails with a check violation (FR-SAF-002). A `REPLAY` run without `replay_from` fails. `config_versions.config_hash` is unique. The engine role cannot `CREATE TABLE`.
- Implement the SQL.

**02.4 Migration 0004 + partition function**
- Tests first: inserting into `market.events` for a day without a partition fails. After `ensurePartitions(aheadDays=3)` inserts succeed for today..today+3. PK uniqueness holds. The dedup unique index rejects a duplicate `(received_at, source, source_event_key)`.

**02.5 Partition manager**
- Tests first: `ensurePartitions` is idempotent. `dropExpiredPartitions(retentionDays)` drops only partitions entirely older than the cutoff, never today's. With `archive` enabled, it calls the archiver callback with the partition name before dropping (the callback is mocked here; the real archiver comes in Phase 22/24).
- Implement for daily and monthly granularity.

**02.6 Kysely types + connection**
- Implement `db:codegen` (kysely-codegen against the local DB) and commit `types.generated.ts`. Connection factory: pool size from config, `statement_timeout` 5 s, `application_name=paperbot-engine`.
- Test: a type-level test that a query on `strategy.runs` is typed.

**02.7 Strategy registry at bootstrap**
- Tests first (integration): first boot inserts the strategy, version and config version. A second boot with the same config reuses them (same IDs). A changed config inserts a new config version.
- Implement: the upserts in one transaction.

## 12. Acceptance Criteria
1. `pnpm db:migrate` on a fresh DB creates all objects in 0001–0004.
2. A LIVE-mode run insert is impossible at the DB level.
3. Partitions are created ahead, and expired ones are dropped (tests).
4. The engine boot registers strategy/config versions idempotently.
5. Integration tests run in CI (Testcontainers).

## 13. Tests
- Data/integration: migrations, constraints, partitions, registry.
- Unit: partition name/range calculation (pure).

## 14. Failure Cases
- DB unreachable at boot → retry with backoff for 60 s, then exit non-zero (the supervisor restarts it).
- Missing partition at insert → the error surfaces as `DB_PARTITION_MISSING`. The persister (Phase 04) will call `ensurePartitions` and retry.
- Supabase lacks role-creation privileges → the migration logs a notice and skips role creation (documented in the verification note).

## 15. Observability
Log migration versions applied at boot (`CONFIG_LOADED`, `PARTITION_MAINTENANCE` system events can be written from now on).

## 16. Security
- Engine runs with the `paperbot_engine` role. Migrations use the owner role.
- RLS deny-all. No objects in `public`.
- Connection strings never logged (redaction).

## 17. Verification
```bash
pnpm dev:db && pnpm db:migrate && pnpm db:partitions
pnpm test:integration --filter @paperbot/db
psql "$DATABASE_OWNER_URL" -c "insert into strategy.runs(mode, ...) values ('LIVE', ...)"   # must fail
```

## 18. Commit Strategy
1. `feat(db): add testcontainers harness and migration runner`
2. `feat(db): add schemas, ops and strategy core migrations`
3. `feat(db): add partitioned market events table`
4. `feat(db): add partition manager`
5. `feat(db): add kysely types and connection factory`
6. `feat(engine): register strategy and config versions at boot`
7. `ci: run integration tests`

## 19. Definition Of Done
- [ ] Acceptance criteria met. Global DoD satisfied.
- [ ] `types.generated.ts` matches the migrated schema (CI check: regenerate + diff).
