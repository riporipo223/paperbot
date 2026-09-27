# Contributing

These rules apply to humans and AI coding agents alike. AI agents must also follow [CLAUDE.md](CLAUDE.md).

## 1. Source-of-truth hierarchy

```text
Product requirements (docs/01, docs/02)
        ↓
System architecture (docs/03, docs/05, docs/06, docs/09)
        ↓
Technical specifications (docs/04, docs/07, docs/08, docs/26)
        ↓
Domain specifications (docs/10–19, docs/20–23, docs/27–29)
        ↓
Build-phase documents (build/phase-*.md)
        ↓
Implementation
        ↓
Tests
```

A higher level wins over a lower one. If implementation conflicts with documentation, **do not silently accept the conflict**:
1. Identify it (file, section, behaviour).
2. Decide whether the doc or the code should change. If it is architectural, ask the operator.
3. Record the decision as an ADR in [docs/32-architecture-decision-records.md](docs/32-architecture-decision-records.md), and update the affected docs in the same change.

## 2. Test-driven development (mandatory)

For every task:
1. **Red**: write a failing test for the next behaviour. Reference the requirement ID in the test name (e.g. `FR-EXE-004`).
2. **Green**: write the minimum code to pass.
3. **Refactor**: improve the design with the tests green.
4. **Commit**.

Never skip, disable, or weaken a test to make CI green. Flaky tests are bugs.

## 3. Commits

- Small and logical: one unit of change per commit, tests and code together.
- Message: short, imperative, English, with a conventional prefix: `feat(scope): …`, `fix(scope): …`, `test(scope): …`, `refactor(scope): …`, `docs: …`, `chore: …`, `ci: …`, `perf: …`, `build: …`.
- Subject line ≤ 72 characters. A body only when the *why* is not obvious.
- No secrets. No generated junk. Lockfile changes only with dependency changes.

## 4. Code rules (enforced by lint/CI where possible)

- TypeScript strict. No `any` without a justification comment.
- No `Date.now()`, `new Date()`, or `Math.random()` in domain code. Use the injected `Clock`/`Rng`.
- No floats for money. Use `bigint` base units / `decimal.js`.
- `process.env` only in `apps/engine/src/bootstrap/**` and the dashboard's server config.
- Modules import each other only through `modules/<name>/index.ts`.
- External payloads are validated with zod at the boundary.
- No signing, keypairs, seed phrases, or transaction-sending APIs. `pnpm check:safety` must pass.
- Untrusted strings (token names/symbols/URIs) are never rendered as HTML and never interpolated into prompts outside fenced data blocks.

## 5. Pull requests

- Reference the phase and task IDs (e.g. "Phase 07, tasks 07.2–07.3").
- Include the verification evidence (commands run, results).
- List any deviation from the docs and the ADR that records it.
- CI must be green. A PR touching `modules/execution`, `modules/portfolio`, `modules/risk` or `modules/economics` needs particularly careful review (money paths).

## 6. Documentation changes

- Keep document headers (Status/Upstream/Downstream/Used by phases) accurate.
- New config keys: update [docs/26-configuration-reference.md](docs/26-configuration-reference.md), the zod schema and the YAML defaults together.
- New provider facts: update [docs/27-data-provider-reference.md](docs/27-data-provider-reference.md) with the verification status and source.
- Update [DOCUMENTATION-INDEX.md](DOCUMENTATION-INDEX.md) when adding or renaming documents.
