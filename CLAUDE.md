# CLAUDE.md — Work Logbook

## Project

Personal web app for capturing achievements (work, learning, side projects, feedback) via a weekly email prompt, with AI that structures entries and generates STAR stories and period summaries on demand.

**`SPEC.md` is the source of truth.** Read the relevant sections before starting any task. If this file and `SPEC.md` disagree, `SPEC.md` wins — and point out the conflict.

The owner, Kitty, is using this project to learn agentic, spec-driven development and to build a portfolio-grade repo. Briefly explain key decisions and trade-offs; keep changes small and reviewable.

## How we work

1. **One phase at a time** (`SPEC.md` §11). Don't build features from later phases or from the out-of-scope list (§3).
2. **Plan first** for anything beyond a trivial fix: list the files you'll touch, the approach, and the tests. Wait for approval.
3. **Spec gaps:** if the spec is unclear, incomplete or contradicts what the code needs, stop and ask. Don't fill gaps with assumptions. After the decision, update `SPEC.md` in the same change.
4. **Tech decisions** get an ADR in `docs/adr/NNNN-short-title.md` (context, decision, alternatives, consequences).
5. **Acceptance criteria** in `SPEC.md` are the checklist. Tick them off in the spec when implemented and tested.

## Accuracy rules

- Don't invent library APIs, config options or CLI flags. Before using an unfamiliar API, or after any version change, check the current official docs. If you couldn't verify something, say so.
- Ask before adding a dependency: say why, and what the alternative is. Record the chosen version in the ADR.
- When reporting results, distinguish what you ran and saw from what you expect.

## Commands

_To be filled in during Phase 0._

```
install:    TBD
dev:        TBD
lint:       TBD
typecheck:  TBD
test:       TBD
e2e:        TBD
db migrate: TBD
db seed:    TBD
```

## Code conventions

- TypeScript strict mode. No `any` without a comment explaining why.
- Validate all external input with Zod: request bodies, form data, env vars, AI output.
- **Every query on user-owned data is scoped by `user_id`**, even in single-user mode.
- Enums (`category`, `impact_type`, `purpose`, statuses) are defined once in `src/domain` and reused by DB, schemas and UI.
- Business logic lives in `src/server/services`, not in components or route handlers.
- `entries.raw_text` is never modified after creation.
- Pure logic (skill normalisation, ISO-week calculation, token handling) lives in `src/domain` and is unit-tested.

## AI module rules

- All AI calls go through `LlmProvider` in `src/ai`. Never import a provider SDK anywhere else.
- Prompt templates live in `src/ai/prompts`, each with an `id` and `version`. Any change to a prompt bumps its version.
- Always validate AI output against its Zod schema, even when the provider claims schema-constrained output.
- Send only the fields listed in `SPEC.md` §7. Never send the user's email or name.
- Prompts must forbid invented facts and numbers; missing info becomes a question, not a guess.
- The app must work with AI disabled.

## Security and privacy

- Secrets only in environment variables. Update `.env.example` when adding a variable.
- Store token hashes only, never raw tokens.
- Don't log entry text or prompt contents; log IDs and error types.
- No personal data in fixtures, seeds, tests or commit messages — the repo may become public.

## Testing

- Unit tests for domain logic, schema validation and AI response handling.
- Tests use `MockProvider` and a fake email sender. **Tests never call a real LLM or send a real email.**
- Every acceptance criterion maps to at least one test.
- Run lint, typecheck and tests before saying a task is done.

## Definition of done

- [ ] Acceptance criteria met and covered by tests
- [ ] Lint, typecheck and tests pass locally and in CI
- [ ] `SPEC.md` updated if behaviour or scope changed; ADR added for tech decisions
- [ ] README updated if setup or commands changed
- [ ] Conventional commit message (`feat:`, `fix:`, `docs:`, `chore:`, `test:`, `refactor:`)
