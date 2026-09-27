# Phase 24 — Deployment

| Field | Value |
|---|---|
| Milestone | M6 Production paper |
| Depends on | 23 |
| Size | M |
| Requirements | NFR-SEC-001..004, NFR-REL-001/002, [25](../docs/25-infrastructure-deployment.md) |

## 1. Objective
Deploy the engine (Docker) and database to the chosen host and the dashboard to Vercel, with secrets management, TLS, backups, graceful restarts, and a runbook. The resulting production paper environment runs continuously in SIMULATION mode.

## 2. Context (read first)
- [25](../docs/25-infrastructure-deployment.md) (entire)
- [21](../docs/21-security-spec.md) §4, §8
- [32](../docs/32-architecture-decision-records.md) ADR-0018, DP-02, DP-03, DP-08

## 3. Dependencies
Phase 23 DONE. **Operator decisions DP-02 (DB hosting) and DP-03 (engine host) are required.** If they are undecided, use the documented defaults (VPS + Docker Compose + self-hosted Postgres) and record that.

## 4. Inputs
Production-ready code, the Dockerfile requirements, and the operator-provided host access and domain (the agent never creates paid accounts itself).

## 5. Outputs
- `apps/engine/Dockerfile`, `deploy/docker-compose.prod.yml`, `deploy/Caddyfile`, `deploy/backup/`, `apps/engine/src/migrate.ts` (container migration entry).
- The GitHub Actions release workflow: build → safety scan → push the image to GHCR (tag = git SHA).
- `docs/runbooks/{deploy.md,rollback.md,restore-backup.md,rotate-secrets.md,incident.md}`.
- Vercel project configuration notes (env var names only).
- A running environment: engine `/ready` = 200, and the dashboard reachable behind login.

## 6. Files To Create
```text
apps/engine/Dockerfile
apps/engine/.dockerignore
apps/engine/src/migrate.ts
deploy/docker-compose.prod.yml
deploy/Caddyfile
deploy/backup/{Dockerfile,backup.sh}
deploy/engine.env.example            # names only
.github/workflows/release.yml
docs/runbooks/{deploy.md,rollback.md,restore-backup.md,rotate-secrets.md,incident.md}
```

## 7. Files To Modify
- [25](../docs/25-infrastructure-deployment.md) (record the chosen options, region, and actual costs)
- [32](../docs/32-architecture-decision-records.md) (ADR-0018 → Accepted with the choice. DP-02/03/08 resolved.)
- `config/env/production.yaml`

## 8. Database Changes
Apply all migrations to production. Create the `paperbot_engine` role with least privilege. On Supabase: verify the schemas are not exposed and RLS is enabled.

## 9. API Changes
None.

## 10. Environment Variables
All production secrets set in the host secret store / env file (chmod 600) and in Vercel. **None are committed.** Verify the forbidden-env guard passes in production.

## 11. Implementation Tasks

**24.1 Dockerfile**
- Tests first: a CI job builds the image and runs a container smoke test (`/health` 200 with the test config and a throwaway Postgres service). `check:safety` runs inside the build and fails it on violations. The image runs as non-root (inspect test).

**24.2 Migration entrypoint**
- Tests first: `node dist/migrate.js` applies pending migrations using `DATABASE_OWNER_URL` and exits non-zero on failure.

**24.3 Production compose + Caddy + backups**
- Implement per [25](../docs/25-infrastructure-deployment.md) §5. The backup container runs a nightly `pg_dump -Fc` to object storage with 14-day retention.
- Verification: a restore test into a scratch DB, recorded in the runbook.

**24.4 Release workflow**
- Implement: on tag or manual dispatch → build, test, push the image, and output the digest.

**24.5 Host setup (operator + agent)**
- The runbook covers chrony, unattended upgrades, the firewall, Docker, and a non-root deploy user. The agent writes the steps. The operator executes the steps that need credentials.

**24.6 Dashboard on Vercel**
- Configure the env var names. Deploy from `main`. Verify the CSP, login, and WS connectivity to the engine origin.

**24.7 First production start**
- Start the engine. Verify the startup banner `MODE=SIMULATION`, `/ready`, streams connected, and the System page is green. Start a LIVE_FEED SIMULATION run with config 0.1.0 (or the operator's chosen version).

**24.8 Graceful restart drill**
- Restart the engine container during an open position (or a fixture-induced one). Verify the recovery report, the position resumes, and there is no ledger imbalance.

## 12. Acceptance Criteria
1. Engine and dashboard are deployed. TLS is valid. The dashboard requires login.
2. Backups run, and one restore was tested.
3. The restart drill passes.
4. No secrets are in the repo, image, or logs (scan the image with gitleaks/trivy secret scanning, and grep the logs).
5. The runbooks exist and were followed once.

## 13. Tests
Container smoke test, migration entrypoint test, and the manual drills (recorded).

## 14. Failure Cases
- Host lacks resources → document the observed usage and upgrade the host size.
- WS blocked by the platform → check the proxy/Caddy config for WebSocket upgrade headers.

## 15. Observability
Log drain (optional) configured. Uptime check on `/health` (optional). Alert webhook configured if desired.

## 16. Security
Least-privilege DB role. The firewall allows only 22/80/443. SSH keys only. Secrets are in the env file with chmod 600 or the platform secret store. Rotation procedure documented.

## 17. Verification
The evidence in `build/verification/phase-24.md`: URLs (without secrets), banner log lines, `/ready` output, backup listing, restore test output, and restart drill notes.

## 18. Commit Strategy
1. `build: add engine dockerfile and migration entrypoint`
2. `build: add production compose, caddy and backups`
3. `ci: add release workflow`
4. `docs: add deployment runbooks`
5. `docs: record hosting decisions and phase 24 verification`

## 19. Definition Of Done
- [ ] Acceptance criteria met. Global DoD satisfied.
- [ ] Milestone M6 marked.
