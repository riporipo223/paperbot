# Development

> The tooling described here is created in **Phase 00**. Until then this file describes the target setup. Authoritative detail: [docs/24-development-environment.md](docs/24-development-environment.md).

## Prerequisites

- Node.js active LTS (exact version in `.nvmrc`), with corepack enabled
- pnpm (version pinned in `package.json` → `packageManager`)
- Docker + Docker Compose (local Postgres, Testcontainers)
- Optional: gitleaks

## First-time setup

```bash
corepack enable
pnpm install
cp .env.example .env.local          # fill in values; NEVER add private keys or seed phrases
pnpm dev:db                         # start local Postgres
pnpm db:migrate
pnpm verify                         # format, lint, typecheck, safety scan, tests
```

## Everyday commands

| Command | Purpose |
|---|---|
| `pnpm dev:engine` | Run the engine (watch mode) |
| `pnpm dev:dashboard` | Run the dashboard |
| `pnpm test` | Unit, contract and in-memory scenario tests |
| `pnpm test:integration` | Integration tests (Docker required) |
| `pnpm test:e2e` | Playwright end-to-end tests |
| `pnpm check:safety` | Static scan: no signing, no secrets, no banned APIs |
| `pnpm verify` | Everything CI runs |
| `pnpm replay -- --config <id> --from <iso> --to <iso>` | Replay stored events through the pipeline |
| `pnpm replay:verify -- --run <id>` | Determinism check against a recorded run |
| `pnpm replay:study -- ...` | Parameter/sensitivity study |
| `pnpm trace:audit -- --run <id>` | Decision trace completeness |
| `pnpm db:partitions` | Partition maintenance |
| `pnpm fixtures:record -- <recorder>` | Record real provider fixtures (manual, real network) |

## Configuration

- Engine config: `config/base.yaml` + `config/env/<env>.yaml`
- Strategy configs (immutable, versioned): `config/strategies/smart-money-wave/<semver>.yaml`
- Secrets: environment variables only
- Reference: [docs/26-configuration-reference.md](docs/26-configuration-reference.md)

## Secrets policy (short)

Allowed: provider API keys, DB URLs, API/session tokens (see `.env.example`). **Forbidden: private keys, seed phrases, keypair files.** The engine refuses to start if it detects them.

## Tests without Docker

Unit, contract and in-memory scenario tests run without Docker. Integration tests (Testcontainers) run in CI. When working in an environment without Docker, say so in the phase verification note.
