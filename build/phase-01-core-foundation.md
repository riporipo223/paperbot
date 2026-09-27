# Phase 01 — Core Foundation (`@paperbot/core`, `@paperbot/config`, engine bootstrap guards)

| Field | Value |
|---|---|
| Milestone | M0 Foundation |
| Depends on | 00 |
| Size | M |
| Requirements | FR-CFG-001..005, FR-SAF-001, FR-SAF-002 (config + bootstrap layers), FR-SAF-004, NFR-TIME-001/002, NFR-DATA-001 |

## 1. Objective
Implement the pure primitives that everything else depends on (time, clock, scheduler, RNG, IDs, money, results, error codes, event envelope, data quality, enums), plus the full configuration system and the engine's startup safety guards (mode guard, forbidden-env guard). After this phase, time, money and randomness have exactly one correct way to be used.

## 2. Context (read first)
- [04](../docs/04-technical-spec.md) §4–5, §7–8
- [09](../docs/09-real-time-data-architecture.md) §2–3, §5, §7
- [26](../docs/26-configuration-reference.md) (entire)
- [21](../docs/21-security-spec.md) §2–4
- [28](../docs/28-decision-logging-spec.md) §6 (reason code registry)
- ADR-0014, ADR-0016

## 3. Dependencies
Phase 00 DONE.

## 4. Inputs
Workspace skeleton with lint and safety rules.

## 5. Outputs
- `@paperbot/core`: time (`EpochMicros`, `SystemClock`, `SimulatedClock`), `Scheduler`, seeded `Rng` with `fork`, `IdGenerator` (UUIDv7), money (`Lamports`, `RawAmount`, decimal helpers), `Result`, error/reason code registry, `EventEnvelope` type + payload schema registry, `DataQuality` + worst-of, all shared enums (zod + TS), and branded address types with base58 validation.
- `@paperbot/config`: full zod schemas for the strategy and engine config ([26](../docs/26-configuration-reference.md)), YAML loader with overlay order, canonical JSON + SHA-256 hashing, cross-field validation, deep freeze.
- `config/base.yaml`, `config/env/{development,test,production}.yaml`, `config/strategies/smart-money-wave/0.1.0.yaml` with the documented defaults.
- Engine `bootstrap/`: `env-guard.ts` (forbidden env detection), `mode-guard.ts`, a logger factory (pino with redaction), and a `main.ts` that loads config, runs the guards, and logs the startup banner, then exits (no modules yet).

## 6. Files To Create
```text
packages/core/src/
  time/{epoch.ts,clock.ts,system-clock.ts,simulated-clock.ts,scheduler.ts,index.ts}
  rng/{rng.ts,xoshiro128.ts,index.ts}
  ids/{uuidv7.ts,index.ts}
  money/{lamports.ts,raw-amount.ts,decimal.ts,index.ts}
  result.ts
  errors/{codes.ts,domain-error.ts,index.ts}
  events/{envelope.ts,event-types.ts,schema-registry.ts,index.ts}
  quality.ts
  enums.ts
  addresses.ts
  canonical-json.ts
  __tests__/...(one test file per module)
packages/config/src/
  schema/{strategy.ts,engine.ts,index.ts}
  loader.ts  hash.ts  validate.ts  index.ts
  __tests__/{schema.test.ts,loader.test.ts,hash.test.ts,validate.test.ts}
config/base.yaml
config/env/development.yaml  config/env/test.yaml  config/env/production.yaml
config/strategies/smart-money-wave/0.1.0.yaml
apps/engine/src/bootstrap/{env-guard.ts,mode-guard.ts,logger.ts,load-config.ts,banner.ts}
apps/engine/src/bootstrap/__tests__/{env-guard.test.ts,mode-guard.test.ts,logger-redaction.test.ts}
```

## 7. Files To Modify
- `apps/engine/src/main.ts` (bootstrap only)
- `eslint.config.js` (allowlist `system-clock.ts` for `Date`/`performance`; allow `process.env` only in `apps/engine/src/bootstrap/**`)
- [24](../docs/24-development-environment.md) §2 (versions: zod, decimal.js, pino, yaml, fast-check)

## 8. Database Changes
None.

## 9. API Changes
None.

## 10. Environment Variables
Read by `bootstrap/load-config.ts`: `PAPERBOT_ENV`, `PAPERBOT_STRATEGY_CONFIG`, and the secrets listed in [24](../docs/24-development-environment.md) §4 (only presence is checked here; values are passed to later modules).

## 11. Implementation Tasks

**01.1 EpochMicros and clocks**
- Tests first: `SystemClock.now()` is monotonic non-decreasing across 10k calls and has µs resolution. `SimulatedClock.advanceTo(t)` rejects going backwards. `monotonicMicros()` differences are ≥ 0.
- Implement per [04](../docs/04-technical-spec.md) §5 (`SystemClock` = `performance.timeOrigin + performance.now()` in µs, rounded to integer).

**01.2 Scheduler**
- Tests first: with `SimulatedClock`, tasks fire in timestamp order when advancing. Tasks scheduled for the same time fire in insertion order. `every()` repeats. Cancel works. Advancing past several due tasks fires all of them in order *before* returning.
- Implement: min-heap scheduler bound to a `Clock`. The live implementation uses `setTimeout` against `SystemClock`, with drift correction.

**01.3 Seeded RNG with fork**
- Tests first: the same seed gives the same sequence. `fork('a')` is independent of draws on the parent and of `fork('b')`. Uniformity smoke test (chi-square on 100k draws, loose bound).
- Implement: xoshiro128** (or splitmix64 → xoshiro). `fork(label)` = hash(seed, label) → new generator.

**01.4 UUIDv7 IdGenerator**
- Tests first: IDs are time-ordered for increasing clock values. The injected clock/RNG make them deterministic in tests. The format validates.
- Implement: UUIDv7 from `Clock` + `Rng`.

**01.5 Money primitives**
- Tests first (property-based with fast-check): lamports ↔ SOL decimal conversions round-trip. `RawAmount` with decimals → decimal string round-trips. Multiplication/division helpers floor correctly. There is no path from bigint to JS `number` for amounts (type-level test).
- Implement: branded bigint types, decimal.js wrappers (precision 40, ROUND_DOWN default for amounts).

**01.6 Result and error codes**
- Tests first: every code in `codes.ts` is unique and namespaced. The reason code families in [28](../docs/28-decision-logging-spec.md) §6 are all present.
- Implement: the registry as `const` objects + TS unions.

**01.7 Enums**
- Tests first: the enum value lists equal the documented sets (snapshot against the values in [07](../docs/07-database-schema.md) CHECK constraints, as a hard-coded expected list in the test).
- Implement: zod enums + TS types for `RunMode`, `RunSource`, `RunStatus`, `DecisionAction`, `RiskOutcome`, `OrderStatus`, `OrderSide`, `OrderIntent`, `OrderFailureReason`, `PositionStatus`, `WaveState`, `WalletStatus`, `WalletSource`, `PoolType`, `DataQuality`, `SubjectKind`, `SourceId`, `AiAgent`, `AiStatus`, `AiEffect`.

**01.8 Addresses**
- Tests first: base58 validation (valid 32-byte keys accepted, wrong length/characters rejected). Signatures (64-byte) are validated separately.
- Implement: branded types + zod refinements (use a small base58 decoder; no signing libraries).

**01.9 DataQuality**
- Tests first: `worstOf` severity order `UNKNOWN > STALE > DEGRADED > FRESH`. `qualityFromAge(age, maxAge, streamHealthy, isFallback)` truth table per [09](../docs/09-real-time-data-architecture.md) §7.

**01.10 EventEnvelope and schema registry**
- Tests first: registering `(type, version, schema)`. Validating an envelope with an unknown type fails. Payload validation errors surface as a `Result` error. The canonical JSON of an envelope is stable.
- Implement: the envelope type per [09](../docs/09-real-time-data-architecture.md) §3 and the event type list per §5 (payload schemas are added by later phases; the registry is empty but typed).

**01.11 Canonical JSON + hashing**
- Tests first: key order independence, number normalization, and a stable SHA-256 for a fixture object.

**01.12 Config schemas**
- Tests first: the default YAML files validate. Each cross-field rule in [26](../docs/26-configuration-reference.md) §4 has a failing fixture and the expected error code. `run.mode: LIVE` → `SAF_MODE_NOT_ALLOWED`. Unknown keys are rejected (`.strict()`).
- Implement: zod schemas mirroring [26](../docs/26-configuration-reference.md) §2–3 exactly.

**01.13 Config loader**
- Tests first: overlay order (base → env → strategy). Env-var secrets are injected but excluded from the strategy hash. The config is deep-frozen (mutation throws). The strategy config hash is stable across key reordering.
- Implement: `loadConfig({env, strategyPath, secrets})` → `{engine, strategy, strategyHash, engineHash, secrets}`.

**01.14 Forbidden-env guard**
- Tests first: each pattern in [21](../docs/21-security-spec.md) §4 triggers `SAF_FORBIDDEN_ENV` (name patterns, 64-int JSON array value, base58-64-byte value, 12-word mnemonic-like value). The allowlisted names (`WS_TOKEN_SECRET`, `SESSION_SECRET`) pass. The error message contains the variable name but **not** the value.
- Implement: `assertNoForbiddenEnv(env)`. Include a compact BIP-39 wordlist check (a wordlist file used only for detection). Add `env-guard.ts`, its wordlist, and its tests to the safety scanner's explicit allowlist (detection code legitimately contains these patterns). Record the allowlist entry in the verification note.

**01.15 Mode guard**
- Tests first: SIMULATION passes. SHADOW fails unless `features.shadow_enabled`. LIVE always fails.

**01.16 Logger with redaction**
- Tests first: a log line containing `apiKey`, `authorization`, `token`, or a URL with `api-key=` is redacted in the output.
- Implement: `createLogger({module})` over pino with redact paths and the base fields from [22](../docs/22-observability-spec.md) §2.

**01.17 Engine main (bootstrap only)**
- Tests first: `bootstrap()` with a valid test config returns a started context and logs the banner containing `MODE=SIMULATION`. With a forbidden env it throws before logging any config values.
- Implement: `main.ts` → `bootstrap()` → exit 0 (placeholder until modules exist).

## 12. Acceptance Criteria
1. All primitives above exist with tests. Domain coverage ≥ 90% for `packages/core` and `packages/config`.
2. `pnpm dev:engine` (with test config) prints the startup banner and exits cleanly.
3. `PRIVATE_KEY=abc pnpm dev:engine` exits non-zero with `SAF_FORBIDDEN_ENV` naming `PRIVATE_KEY`.
4. `run.mode: LIVE` in a strategy file is rejected at load.
5. The config hash of `0.1.0.yaml` is stable and recorded in the verification note.

## 13. Tests
Unit and property tests listed per task. No integration tests.

## 14. Failure Cases
- Invalid YAML → `CFG_PARSE_ERROR` with file and line.
- Missing required secret for an enabled feature (e.g. `ai.mode != OFF` without `ANTHROPIC_API_KEY`) → `CFG_MISSING_SECRET` at bootstrap. With `ai.mode = OFF` it is not required.
- Simulated clock going backwards → throws (programming error).

## 15. Observability
Logger available. The startup banner includes the config hashes, env, and mode.

## 16. Security
The env guard and mode guard are mandatory at startup. Secrets never appear in logs (redaction test).

## 17. Verification
```bash
pnpm verify
PAPERBOT_ENV=test pnpm dev:engine          # banner
PRIVATE_KEY=x PAPERBOT_ENV=test pnpm dev:engine; echo $?   # non-zero
```

## 18. Commit Strategy
1. `feat(core): add epoch time, clocks and scheduler`
2. `feat(core): add seeded rng and uuidv7 ids`
3. `feat(core): add money primitives and canonical json`
4. `feat(core): add enums, addresses, error codes, data quality`
5. `feat(core): add event envelope and schema registry`
6. `feat(config): add strategy and engine config schemas`
7. `feat(config): add loader, hashing and cross-field validation`
8. `feat(engine): add bootstrap guards and logger`
9. `docs: phase 01 verification`

## 19. Definition Of Done
- [ ] Acceptance criteria met.
- [ ] Global DoD satisfied.
- [ ] `config/strategies/smart-money-wave/0.1.0.yaml` hash recorded.
