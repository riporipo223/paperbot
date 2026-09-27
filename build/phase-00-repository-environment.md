# Phase 00 — Repository & Development Environment

| Field | Value |
|---|---|
| Milestone | M0 Foundation |
| Depends on | — |
| Size | S |
| Requirements | FR-SAF-003 (scanner skeleton), NFR maintainability ([03](../docs/03-system-requirements.md) §8) |

## 1. Objective
Create a monorepo skeleton where every later phase can add code with tests, linting, type checking, formatting, safety scanning and CI already enforced. **No application logic.**

## 2. Context (read first)
- [04-technical-spec.md](../docs/04-technical-spec.md) §1–4, §6, §9
- [24-development-environment.md](../docs/24-development-environment.md)
- [21-security-spec.md](../docs/21-security-spec.md) §2, §4
- [23-testing-strategy.md](../docs/23-testing-strategy.md) §1, §6
- [CONTRIBUTING.md](../CONTRIBUTING.md), [CLAUDE.md](../CLAUDE.md)

## 3. Dependencies
None.

## 4. Inputs
The repository containing only documentation (`docs/`, `build/`, root markdown files).

## 5. Outputs
- pnpm workspace with empty packages `@paperbot/core`, `@paperbot/config`, `@paperbot/db`, `@paperbot/api-contract`, `@paperbot/engine` (`apps/engine`), each with a placeholder `index.ts` and one trivial passing test. `apps/dashboard` is **not** created yet (Phase 19).
- Shared TS config (strict), ESLint flat config with the custom bans, Prettier, dependency-cruiser rules.
- `scripts/check-safety.ts` (safety scanner) with its own tests.
- `scripts/check-config-immutability.ts`.
- `docker-compose.yml` (Postgres).
- GitHub Actions CI workflow.
- `.env.example`, `.gitignore`, `.nvmrc`, `.editorconfig`.
- Filled "pinned versions" table in [24](../docs/24-development-environment.md) §2 for the tools installed in this phase.

## 6. Files To Create
```text
package.json                     # root: scripts (lint, format, typecheck, test, test:integration, check:safety, verify), packageManager
pnpm-workspace.yaml
.nvmrc  .editorconfig  .gitignore  .env.example  .prettierrc  .prettierignore
tsconfig.base.json               # strict flags from 04 §4
eslint.config.js                 # typescript-eslint strict-type-checked + custom restrictions (see Task 00.4)
.dependency-cruiser.cjs          # rules from 04 §6 (initial subset)
vitest.workspace.ts
docker-compose.yml
packages/core/{package.json,tsconfig.json,src/index.ts,src/__tests__/smoke.test.ts}
packages/config/{...same...}
packages/db/{...same...}
packages/api-contract/{...same...}
apps/engine/{package.json,tsconfig.json,src/main.ts,src/__tests__/smoke.test.ts}
scripts/check-safety.ts
scripts/__tests__/check-safety.test.ts
scripts/__tests__/fixtures/banned/*.ts.txt     # files containing each banned pattern (as .txt so they don't compile)
scripts/check-config-immutability.ts
.github/workflows/ci.yml
build/verification/.gitkeep
```

## 7. Files To Modify
- `docs/24-development-environment.md` §2 (pinned versions)
- `build/PHASE-STATUS.md`

## 8. Database Changes
None. The Postgres container is only defined.

## 9. API Changes
None.

## 10. Environment Variables
`.env.example` exactly as in [24](../docs/24-development-environment.md) §4 (values empty).

## 11. Implementation Tasks

**Task 00.1: Workspace skeleton**
- Test first: `packages/core/src/__tests__/smoke.test.ts` asserts `import { VERSION } from '../index'` is a string (it fails until the file exists).
- Implement: root `package.json` (with `"type": "module"`, `packageManager: pnpm@<pinned>`), `pnpm-workspace.yaml` (`packages/*`, `apps/*`), package skeletons exporting `VERSION`.
- Done when: `pnpm install && pnpm test` passes.

**Task 00.2: TypeScript strict config**
- Test first: add a type-level test file `packages/core/src/__tests__/strict.test-d.ts` using `expectTypeOf` (vitest), which relies on `noUncheckedIndexedAccess` (e.g. array index is `T | undefined`).
- Implement: `tsconfig.base.json` with the strict flags from [04](../docs/04-technical-spec.md) §4 and project references. `pnpm typecheck` = `tsc -b`.
- Done when: `pnpm typecheck` passes and the type test passes.

**Task 00.3: Prettier + EditorConfig**
- Implement: config and `format` / `format:check` scripts.
- Done when: `pnpm format:check` passes on the repo (docs included, or excluded explicitly in `.prettierignore` if formatting would reflow tables; the decision is recorded in the verification note).

**Task 00.4: ESLint with project bans**
- Test first: `scripts/__tests__/eslint-rules.test.ts` runs ESLint programmatically on inline code samples and expects errors for: `Date.now()`, `new Date()` (outside allowlisted files), `Math.random()`, `parseFloat`, `dangerouslySetInnerHTML`, `it.skip`, and imports of `@solana/web3.js`, `bip39`, `ed25519-hd-key`, `tweetnacl`.
- Implement: `eslint.config.js` with `no-restricted-syntax`, `no-restricted-imports`, `no-restricted-properties`. Allowlist `packages/core/src/time/system-clock.ts` for `Date`/`performance`.
- Done when: the rule tests pass and `pnpm lint` passes.

**Task 00.5: Safety scanner**
- Test first: `scripts/__tests__/check-safety.test.ts`. The scanner run on `fixtures/banned/` reports one violation per file (sendTransaction, sendRawTransaction, simulateTransaction, requestAirdrop, Keypair, generateKeyPairSigner, createKeyPairSignerFromBytes, signTransaction, mnemonic/bip39, a 64-int JSON array literal, `id.json` filename). Run on a clean fixture → 0 violations.
- Implement: `scripts/check-safety.ts` scanning `apps/**`, `packages/**`, `scripts/**` (excluding its own fixtures and tests), exiting non-zero with file:line findings. Script `check:safety`.
- Done when: tests pass and `pnpm check:safety` passes on the repo.

**Task 00.6: Dependency-cruiser**
- Test first: a fixture mini-project under `scripts/__tests__/fixtures/depcruise/` with a forbidden import (domain → pg) that depcruise must flag (a test runs depcruise's API against it).
- Implement: `.dependency-cruiser.cjs` with rules: no deep imports across engine modules, `domain/` cannot import `pg|ws|fastify|@anthropic-ai/sdk|@solana/*`, `apps/dashboard` cannot import `apps/engine|@paperbot/db`. Add to `pnpm lint`.
- Done when: the test passes and `pnpm lint` passes.

**Task 00.7: Config immutability check**
- Test first: a unit test with a fake git diff list (modified `config/strategies/x/0.1.0.yaml` → fail; added → pass).
- Implement: the script (reads `git diff --name-status origin/main...HEAD` in CI; accepts a file list in tests).
- Done when: tests pass. The script is wired into CI (not into `verify`, since it needs git history).

**Task 00.8: Docker Compose Postgres**
- Implement: `docker-compose.yml` per [24](../docs/24-development-environment.md) §3 with a pinned image tag. Script `dev:db`.
- Done when: `docker compose config` validates. If Docker is available, `pnpm dev:db` becomes healthy.

**Task 00.9: CI workflow**
- Implement: `.github/workflows/ci.yml` per [24](../docs/24-development-environment.md) §7 (steps that exist so far: install, format, lint, typecheck, safety, test, gitleaks, config immutability).
- Done when: the workflow is syntactically valid (`actionlint` if available) and the pushed branch runs green.

**Task 00.10: Git hooks (optional)**
- Implement: `simple-git-hooks` + `lint-staged` as in [24](../docs/24-development-environment.md) §6.

**Task 00.11: Verification note**
- Write `build/verification/phase-00.md` with command outputs and pinned versions.

## 12. Acceptance Criteria
1. `pnpm install --frozen-lockfile && pnpm verify` exits 0 on a clean clone.
2. Adding `Math.random()` to any engine file makes `pnpm lint` fail.
3. Adding `connection.sendTransaction(` anywhere under `apps/` makes `pnpm check:safety` fail.
4. CI runs on push and passes.
5. No application logic exists yet (only skeletons).

## 13. Tests
- Unit: smoke tests, ESLint rule tests, safety scanner tests, depcruise rule test, config immutability test.
- Integration/E2E: none.

## 14. Failure Cases
- Tool version incompatibilities → pin to versions that work together. Record them in [24](../docs/24-development-environment.md) §2.
- `prettier` reflowing doc tables → exclude `docs/**` and `build/**` from Prettier if needed (record the decision).

## 15. Observability
None yet (logger arrives in Phase 01).

## 16. Security
- `.gitignore` includes `.env*` (except `.env.example`), `*.keypair.json`, `id.json`, `**/wallet*.json`.
- The safety scanner is a required CI step from now on.
- No secrets in the repo. gitleaks runs in CI.

## 17. Verification
Run and paste into `build/verification/phase-00.md`:
```bash
pnpm install --frozen-lockfile
pnpm verify
echo 'export const x = Math.random();' >> apps/engine/src/main.ts && pnpm lint; git checkout apps/engine/src/main.ts   # must fail, then revert
```

## 18. Commit Strategy
1. `chore: init pnpm workspace and package skeletons`
2. `chore: add strict tsconfig and prettier`
3. `chore: add eslint with project safety bans`
4. `chore: add safety scanner with tests`
5. `chore: add dependency-cruiser boundary rules`
6. `chore: add docker compose postgres`
7. `ci: add github actions workflow`
8. `docs: record pinned versions and phase 00 verification`

## 19. Definition Of Done
- [ ] All acceptance criteria met.
- [ ] Global DoD ([31](../docs/31-build-phases.md)) satisfied.
- [ ] `build/PHASE-STATUS.md`: Phase 00 = DONE with the commit SHA.
