# 21 — Security Specification

| Field | Value |
|---|---|
| Status | Draft v0.1 |
| Upstream | [02](02-product-requirements.md) §1, [03](03-system-requirements.md) §7, [05](05-system-architecture.md) |
| Downstream | Every phase ("Security" section), [29](29-future-live-trading-architecture.md) |
| Used by phases | 00 (safety scans), 01 (env guard), 03 (RPC allowlist), 14 (executor), 16 (AI), 18 (API), 19 (auth), 24 (deploy) |

## 1. Threat model (MVP)

| Asset | Threats | Primary controls |
|---|---|---|
| **Real funds** | Accidental live execution, key leakage | No keys exist anywhere. There is no code path to sign or send (structural + CI). LIVE mode is blocked in three layers. |
| Provider API keys (Helius, Anthropic, CoinGecko) | Leakage via logs, frontend bundle, AI prompts, repo | Secrets only in env/secret stores. Pino redaction. Bundle scan. Pre-commit secret scan. |
| Engine API | Unauthorized control (start runs, pause entries), data scraping | Bearer service token, short-lived WS tokens, CORS, rate limits, idempotency |
| Dashboard | Credential stuffing, session theft, XSS via token metadata | argon2id password hash, rate-limited login, HTTP-only SameSite=Strict cookies, CSP, escaping of untrusted strings |
| Decision integrity | Prompt injection via token metadata steering the AI | AI veto-only, enum-only effects, fenced inputs, schema validation |
| Data integrity | Malicious/malformed provider payloads | zod validation at every boundary, dead-lettering, no partial application |
| Database | Exposure via Supabase REST (PostgREST) | Custom schemas not exposed. RLS deny-all. Least-privilege engine role. |
| Supply chain | Malicious npm dependency | Lockfile, `pnpm audit` in CI, minimal deps, Renovate/Dependabot with review, no postinstall scripts from unknown packages (`pnpm` `onlyBuiltDependencies` allowlist) |

## 2. The simulation isolation guarantee

The MVP must make real execution **impossible, not merely disabled**:

1. **No signing capability in the codebase.** Banned identifiers and imports (enforced by `scripts/check-safety.ts` in CI and ESLint `no-restricted-imports`/`no-restricted-syntax`):
   - Imports: the entire `@solana/web3.js` package (the project uses `@solana/kit` only for encoding/decoding helpers), `@solana/kit` signer/keypair and transaction-sending modules (`generateKeyPairSigner`, `createKeyPairSignerFromBytes`, `signTransaction`, `sendAndConfirmTransactionFactory`, `sendTransactionWithoutConfirmingFactory`, and any `*Signer*` export), `bip39`, `ed25519-hd-key`, `tweetnacl` sign APIs, `bs58` decode of 64-byte secrets.
   - RPC method strings: `sendTransaction`, `sendRawTransaction`, `simulateTransaction`, `requestAirdrop`.
   - File patterns: `*.keypair.json`, `id.json`, `**/wallet*.json` in the repo (gitignored and scan-blocked).
2. **Read-only RPC client.** `SolanaReadClient` exposes only the allowlisted methods ([27](27-data-provider-reference.md) §4.4). A generic `call(method)` asserts the allowlist and throws `SAF_RPC_METHOD_BLOCKED` otherwise. Unit-tested.
3. **LIVE blocked in three layers**: config schema (`run.mode` enum accepts `LIVE` only to reject it with `SAF_MODE_NOT_ALLOWED`), bootstrap `ModeGuard`, DB CHECK constraint on `strategy.runs.mode`.
4. **`RealExecutor` is a stub** that throws, and it is not wired into the factory ([16](16-paper-trading-spec.md) §2).
5. **Network egress awareness**: the engine's HTTP clients are constructed only for the configured provider hostnames (allowlist `security.allowed_hosts`). An outbound request to any other host throws. This prevents accidental calls to execution venues (e.g. Jupiter swap endpoints) until Phase 26 explicitly adds read-only quote hosts.

## 3. Mode guard

`bootstrap/mode-guard.ts` runs before any module is constructed:
- `run.mode` must be `SIMULATION` (or `SHADOW` when `features.shadow_enabled`, Phase 26+).
- Asserts no forbidden env vars (§4).
- Asserts `ExecutorFactory` returns `PaperExecutor` for the mode.
- Logs a startup banner: `MODE=SIMULATION • real execution: IMPOSSIBLE (no signer compiled) • executor=PaperExecutor`.

## 4. Secrets policy

| Secret | Where it lives | Never in |
|---|---|---|
| `HELIUS_API_KEY` | Engine host env / secret store | DB, logs, frontend, AI prompts, repo, error messages, URLs in logs (the RPC URL containing the key is redacted) |
| `ANTHROPIC_API_KEY` | Engine env | same |
| `COINGECKO_API_KEY` (optional) | Engine env | same |
| `DATABASE_URL` | Engine env (and CI secrets for integration tests against ephemeral DBs only) | same |
| `ENGINE_API_TOKEN` | Engine env + Vercel server env | browser |
| `WS_TOKEN_SECRET` | Engine env | anywhere else |
| `HELIUS_WEBHOOK_AUTH` (optional) | Engine env + Helius dashboard | same |
| `DASHBOARD_PASSWORD_HASH`, `SESSION_SECRET` | Vercel env | engine, browser |
| `METRICS_TOKEN` | Engine env + scraper | browser |

**Forbidden env guard (FR-SAF-004):** at startup the engine refuses to run if any env var:
- has a name matching `/(PRIVATE|SECRET)_?KEY|SEED|MNEMONIC|KEYPAIR|WALLET_KEY|SIGNER/i`, except the explicit allowlist above (e.g. `WS_TOKEN_SECRET`, `SESSION_SECRET` are allowlisted by exact name);
- has a value that looks like a Solana secret key: a JSON array of 64 integers 0–255, or a base58 string decoding to 64 bytes, or a 12/24-word BIP-39-like phrase (≥ 12 lowercase words from the BIP-39 wordlist).

The error names the variable but never prints the value.

**Logging redaction:** pino `redact` paths for `*.apiKey`, `*.authorization`, `*.token`, `headers.authorization`, `headers.x-cg-demo-api-key`, and URL query strings containing `api-key`. A test asserts that redaction works for each path.

**Frontend bundle scan:** CI greps the built dashboard (`.next/static`) for engine secrets' env names and value patterns. Only `NEXT_PUBLIC_*` variables may be referenced client-side, and the allowlist is `NEXT_PUBLIC_ENGINE_WS_URL`, `NEXT_PUBLIC_APP_ENV`.

**Repo:** `.gitignore` covers `.env*` (except `.env.example`). Pre-commit hook and CI run `gitleaks` (or an equivalent secret scanner).

## 5. API security

- TLS everywhere (the platform terminates TLS; the engine serves HTTP behind it).
- Bearer token comparison is constant-time.
- WS tokens: HS256 JWT with `exp ≤ 5 min`, `aud`, `jti`. The engine keeps a short replay cache of `jti` for the token lifetime.
- CORS: only `DASHBOARD_ORIGIN`. Credentials not allowed cross-origin (not needed).
- Input validation: zod on every body/query/param. Body size limits. Pagination limits (≤ 500).
- Rate limits per token and per IP (`@fastify/rate-limit`).
- Security headers (`@fastify/helmet`).
- Command endpoints require `Idempotency-Key`. Responses are cached per key for 24 h.
- Errors never include stack traces or secrets in production.

## 6. Dashboard security

- CSP: `default-src 'self'; connect-src 'self' <ENGINE_WS_ORIGIN> <ENGINE_HTTP_ORIGIN>; img-src 'self' data:; script-src 'self' (+ nonce for Next.js); frame-ancestors 'none'`.
- No external images or scripts. Token logos are not fetched in the MVP.
- Untrusted strings are rendered as text only ([20](20-dashboard-spec.md) §5.6). `dangerouslySetInnerHTML` is banned by lint.
- The login route has rate limits and lockout after 10 failures per 15 min per IP.

## 7. AI-specific security (prompt injection)

- Untrusted fields are passed inside `<untrusted_data>` blocks with a system instruction that content inside is data, never instructions. Fields are truncated (name/symbol ≤ 64 chars, description ≤ 500), control/bidi characters are stripped, and the text is JSON-encoded.
- AI outputs only affect decisions through the `verdict` enum in VETO mode. Free text is display-only.
- No tools with side effects are given to any model. The only "tool" is the output schema.
- Secrets are never included in prompts. A prompt builder test asserts the known env var values are absent from rendered prompts.

## 8. Database security

- Roles: `paperbot_owner` (migrations), `paperbot_engine` (DML on the six schemas, no DDL), `paperbot_readonly` (optional analytics).
- Supabase: the six schemas are excluded from the API. RLS is enabled with no policies (deny-all) as a backstop. The service role key is never used by the engine (it uses the DB role connection string).
- Backups: platform daily backups (Supabase) or `pg_dump` cron for self-hosted ([25](25-infrastructure-deployment.md)).

## 9. Future live execution boundary

Live trading is **out of scope** and requires a separate architecture and security review ([29](29-future-live-trading-architecture.md)). The minimum preconditions recorded here:
- The signer runs in a separate, isolated service (or hardware/remote signer). It is never in the engine process, and never in any environment that has provider or AI keys.
- The signer enforces its own independent limits (per-tx max, daily max, allowlisted programs, destination allowlist).
- Keys are never in env vars, DB, logs, prompts or source. They live in a KMS/HSM or dedicated signer only.
- Human-approved rollout with kill switch and tiny limits.

## 10. Security tests (summary)

- Safety scan fails on a fixture file containing each banned pattern (tests of the scanner itself).
- Forbidden env guard: each pattern triggers a refusal, and the allowlisted names pass.
- RPC allowlist blocks `sendTransaction`.
- ModeGuard refuses LIVE. The DB rejects a LIVE run insert.
- API: unauthenticated requests are rejected. An expired WS token is rejected. A CORS-disallowed origin is rejected.
- Redaction test for logs.
- Bundle scan in CI.
