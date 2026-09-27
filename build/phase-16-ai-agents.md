# Phase 16 — AI Agent Layer

| Field | Value |
|---|---|
| Milestone | M4 Reproducible |
| Depends on | 15 |
| Size | M |
| Requirements | FR-AI-001..004, FR-DEC-002, FR-WAL-005 |

## 1. Objective
Implement the advisory AI layer behind the `AiAdvisor` port: the LLM client abstraction (Anthropic, Fake, Recorded), prompt registry, zod output schemas, the six agents, budget and circuit breaker, untrusted-input sanitization, and persistence of every call in `strategy.ai_outputs`. Integrate `ENTRY_REVIEWER` into the decision path for VETO mode, with a timeout and deterministic fallback.

## 2. Context (read first)
- [13](../docs/13-ai-agent-architecture.md) (entire)
- [21](../docs/21-security-spec.md) §7
- [14](../docs/14-strategy-engine-spec.md) §3.2 (rules 6–7)
- [26](../docs/26-configuration-reference.md) `ai.*`
- The official Anthropic API documentation (verify the SDK usage, tool-use structured output, model IDs, and pricing at implementation time)

## 3. Dependencies
Phase 15 DONE.

## 4. Inputs
The strategy AI port (Noop), domain data (signals, wallets, trades), and config.

## 5. Outputs
- Migration `0013_ai_outputs.sql` (+ the FK from decisions).
- `modules/ai` per [13](../docs/13-ai-agent-architecture.md) §4.
- The `ai.pricing` table filled in with verified prices, with the verification date recorded in [27](../docs/27-data-provider-reference.md) §8.

## 6. Files To Create
```text
packages/db/migrations/0013_ai_outputs.sql
apps/engine/src/modules/ai/
  index.ts ports.ts service.ts
  llm-client.ts anthropic-client.ts fake-llm-client.ts recorded-llm-client.ts
  prompt-registry.ts prompts/{entry-reviewer@1.0.0.ts,token-context@1.0.0.ts,wallet-classifier@1.0.0.ts,signal-explainer@1.0.0.ts,post-trade-analyst@1.0.0.ts,research@1.0.0.ts}
  schemas/{entry-reviewer.ts,token-context.ts,wallet-classifier.ts,signal-explainer.ts,post-trade-analyst.ts,research.ts}
  budget.ts sanitize.ts input-hash.ts
  agents/{entry-reviewer.ts,token-context.ts,wallet-classifier.ts,signal-explainer.ts,post-trade-analyst.ts,research.ts}
  adapters/ai-output-repository.ts
  __tests__/...
apps/engine/src/cli/ai-research.ts
```

## 7. Files To Modify
- `modules/strategy/ai-port.ts` → `AiAdvisorImpl` using the ai module (the Noop stays for `ai.mode=OFF`).
- `docs/27-data-provider-reference.md` §8 (verified model IDs/pricing).

## 8. Database Changes
`0013_ai_outputs.sql`: `strategy.ai_outputs` + `ALTER TABLE strategy.decisions ADD CONSTRAINT decisions_ai_output_fk FOREIGN KEY (ai_output_id) REFERENCES strategy.ai_outputs(id)`.

## 9. API Changes
None (Phase 18 exposes them).

## 10. Environment Variables
`ANTHROPIC_API_KEY` (required only if `ai.mode != OFF`).

## 11. Implementation Tasks

**16.1 LlmClient port + FakeLlmClient**
- Tests first: the fake returns scripted outputs, delays (on the simulated clock), errors and invalid payloads.

**16.2 Output schemas**
- Tests first: valid, invalid enum, extra fields (rejected via `.strict()`), oversized strings, and wrong types → rejected.

**16.3 Sanitization and fencing**
- Tests first: control/bidi characters stripped, truncation lengths, JSON-encoding inside `<untrusted_data>`. Known secret values never appear in the rendered prompts (inject fake env values and assert absence).

**16.4 Input hashing**
- Tests first: stable `input_hash` across key ordering. Different prompt version → a different hash.

**16.5 Budget + circuit breaker**
- Tests first: cost accumulation from usage × pricing. The daily cap → `BUDGET_EXCEEDED` until the UTC rollover. Per-agent hourly caps. The breaker opens after 5 failures in 5 min.

**16.6 AnthropicClient**
- Tests first (mocked HTTP via undici MockAgent, with recorded response shapes): the forced-tool structured output request (tool `input_schema` generated from zod), parsing the tool input, usage extraction, and timeout via AbortSignal. **No live calls in tests.**
- Implement with the official SDK per the current docs. Host `api.anthropic.com` is in the allowlist.

**16.7 Agents**
- Tests first per agent: trigger conditions, input building (from domain data only; addresses of third parties omitted where specified), and recording of `strategy.ai_outputs` for every status.

**16.8 ENTRY_REVIEWER in the decision path**
- Tests first (scenario with FakeLlmClient): VETO mode + a veto with confidence ≥ threshold → REJECT (AI_VETO). Confidence below → ENTER. Timeout → PROCEED/REJECT/WAIT per config. ADVISORY mode never changes the action. A late result updates the ai_output row, flagged late, and does not revisit the decision.

**16.9 Async agents wiring**
- Tests first: TOKEN_CONTEXT on wave SEEDED. WALLET_CLASSIFIER on score updates (refresh interval). SIGNAL_EXPLAINER on decisions (if enabled). POST_TRADE_ANALYST on position closed. All are non-blocking.

**16.10 RecordedLlmClient (for replay)**
- Tests first: a hit returns the stored output and replays the recorded latency relative to the timeout. A miss → TIMEOUT (or live if configured, marking the run non-reproducible).

**16.11 Research CLI**
- `pnpm ai:research -- --run <id>` → a RESEARCH output stored and printed. Its suggestions are never auto-applied (test: no config version is created).

**16.12 Live smoke (manual)**
- With a real key and `ai.mode=ADVISORY`, run for ≥ 2 h. Record the call counts, statuses, latency, and cost in the verification note.

## 12. Acceptance Criteria
1. All agents record outputs with a status, model, prompt version, hash, latency and cost.
2. VETO semantics and fallbacks are proven by scenarios. ADVISORY has zero effect on decisions (S11 variants).
3. The prompt-injection fixture has no effect on decisions (structural test).
4. The budget cap is enforced.
5. No secrets in prompts (test).

## 13. Tests
Unit (schemas, sanitize, budget, hashing, client with mocks), scenario (decision path), integration (repository).

## 14. Failure Cases
Per [13](../docs/13-ai-agent-architecture.md) §7, each with a test.

## 15. Observability
Metrics: `paperbot_ai_calls_total{agent,status}`, `paperbot_ai_cost_usd_total{agent}`. Logs: `ai.output.recorded`, `ai.call.failed`, `ai.budget.exceeded`.

## 16. Security
Prompt-injection defences. No tools with side effects. The key is only in env. Outputs are stored after validation, and raw excerpts are truncated (≤ 4 KB).

## 17. Verification
```bash
pnpm test --filter @paperbot/engine -- ai scenarios/s11
pnpm test:integration --filter @paperbot/engine -- ai
```

## 18. Commit Strategy
1. `feat(db): add ai outputs table and decision fk`
2. `feat(ai): add llm client port, fake client and schemas`
3. `feat(ai): add sanitization, hashing and budget`
4. `feat(ai): add anthropic client with structured output`
5. `feat(ai): add advisory agents`
6. `feat(ai): integrate entry reviewer veto path`
7. `feat(ai): add recorded client for replay`

## 19. Definition Of Done
- [ ] Acceptance criteria met. Global DoD satisfied.
