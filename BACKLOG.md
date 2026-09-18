# Backlog — Work Logbook

Derived from `SPEC.md` (Draft 0.1, 2026-09-18). Tasks are grouped by the phases in SPEC.md §11 and numbered in a suggested order. Each task is sized for one session and written so it can be picked up without reading the other tasks; where a task builds on earlier work, the description says what it assumes exists.

Before starting a task: read `CLAUDE.md` and the SPEC.md sections named in the task. When a task covers acceptance criteria, tick them in SPEC.md once they are tested.

Conventions used below:

- **ADR** = a decision record in `docs/adr/NNNN-short-title.md` with context, decision, alternatives, consequences and the versions chosen. Verify current docs before deciding.
- **Tests** = unit tests unless stated otherwise. Tests never call a real LLM or send a real email.
- **Spec gap** = something the task needs that SPEC.md does not define. Stop and ask, then update SPEC.md in the same change.
- Open questions Q2–Q5 in SPEC.md §12 are owner decisions, not tasks; Q4 (public or private repo) should be settled before Phase 0 ends.

---

# Phase 0 — Setup

## 1. Scaffold an empty project with a passing test

Goal: A Next.js + TypeScript (strict) project that installs, runs and passes one trivial unit test.

Description: Create the Next.js App Router project in TypeScript strict mode, matching the folder layout in SPEC.md §10 (`src/domain`, `src/server`, `src/ai`, `src/db`, `tests`, `docs/adr`). Pick a unit test runner, record it in ADR 0001 with versions, and add one trivial test (for example a placeholder function in `src/domain`) that passes. Fill in the `install`, `dev` and `test` commands in CLAUDE.md and add a README skeleton with the same commands.

## 2. Lint and typecheck

Goal: `lint` and `typecheck` commands that pass on the empty project.

Description: Add an ESLint configuration for TypeScript and Next.js and a `typecheck` script (`tsc --noEmit`), both wired as npm scripts. Record the tools and versions in the tooling ADR. Fill the `lint` and `typecheck` lines in CLAUDE.md and README. A formatter is not in the spec: ask before adding one.

## 3. CI on GitHub Actions

Goal: Every push runs lint, typecheck and tests on GitHub Actions (SPEC.md §8, Quality).

Description: Add a workflow that installs dependencies from the lockfile, then runs lint, typecheck and unit tests. Confirm it is green on the current empty app and add the status badge to README. Do not add a database service yet; the migrations task adds it if needed.

## 4. ADR: ORM and migrations

Goal: Decide the ORM or query builder and migration tool for PostgreSQL.

Description: Compare candidates against SPEC.md §6 (UUID ids, enums defined once in code and reused by the DB, composite primary keys, unique constraints) and §9. Verify current versions and APIs in official docs, then write the ADR. Do not install anything yet; the database schema task in Phase 1 installs it.

## 5. ADR: magic link auth library

Goal: Decide how magic link login (SPEC.md F1) is implemented.

Description: Evaluate auth libraries for Next.js App Router that support email magic links, single-use links with a 15-minute expiry, and an email allowlist with sign-up disabled. Consider how sessions will protect all pages except login, the token capture page and the health check. Write the ADR and record versions.

## 6. ADR: transactional email provider

Goal: Decide the provider for login links and weekly prompt emails.

Description: Compare providers on free or cheap tier, EU data handling, API simplicity and deliverability, checking pricing and API in current official docs. Write the ADR and list the environment variables the provider will need; they are added to `.env.example` when the email module is built.

## 7. ADR: hosting and managed Postgres

Goal: Decide where the app and database run (SPEC.md D11, §8 Privacy).

Description: Compare managed PaaS options with managed Postgres in an EU region on a free or cheap tier. Weigh cost, ops effort, Next.js support and how secrets are managed. Write the ADR and note anything that constrains the scheduler decision (task 8).

## 8. ADR: scheduler mechanism

Goal: Decide how the daily job that sends weekly prompts is triggered (SPEC.md F3, Scheduling).

Description: Compare platform cron from the chosen hosting against an external scheduler. Requirements: runs at least daily in UTC, calls a secret-protected endpoint, and is safe to re-run. Write the ADR.

## 9. ADR: end-to-end test tool

Goal: Decide the browser end-to-end test tool.

Description: Compare e2e tools for a Next.js app on setup effort, CI cost and mobile viewport support (needed for the capture page). Write the ADR. Installing it and writing the first e2e test is task 26.

## 10. Environment variable schema and `.env.example`

Goal: All configuration is read through one Zod-validated env module.

Description: Create an env module that parses `process.env` with Zod and fails fast with a clear message on missing or invalid values. Start with the variables known so far (database URL, allowlisted email, job secret, app base URL) and keep `.env.example` in sync with placeholder values. Add unit tests for the schema and update the README setup section.

---

# Phase 1 — Core

## 11. Domain enums and types

Goal: Every enum in SPEC.md §6 is defined once in `src/domain`.

Description: Create the `category`, `impact_type`, `purpose`, entry `status`, entry `source`, AI proposal `status` and generated output `type` enums as const arrays with Zod schemas and TypeScript types in `src/domain`. Export them for later reuse by the DB schema, validation and UI. Add tests that the values match the spec exactly.

## 12. Skill name normalisation

Goal: A pure, tested `normalizeSkillName` function (SPEC.md F5).

Description: Implement lowercase, then strip whitespace, `-`, `_` and `.`, in `src/domain`. Cover the spec example ("Power BI", "PowerBI" and "power-bi" all become `powerbi`), non-ASCII letters, and empty or whitespace-only input. Spec gap: whether an empty normalised result is rejected is not defined; ask.

## 13. Database schema and migrations

Goal: All tables from SPEC.md §6 exist via migrations.

Description: Using the ORM chosen in the ORM ADR, define `users`, `entries`, `skills`, `entry_skills`, `weekly_prompts`, `ai_proposals`, `generated_outputs` and `generated_output_entries` with UUID ids, `user_id`, `created_at` and `updated_at`, the unique constraints listed in §6, and enums sourced from `src/domain`. Generate the first migration, add the `db migrate` command to CLAUDE.md and README, and document local Postgres setup. Add a Postgres service to CI if migrations run there.

## 14. Development seed script

Goal: `db seed` creates the owner user and a few sample entries locally.

Description: Add a seed script that inserts the allowlisted user from env plus a small set of invented entries and skills. No personal data: all content is fictional. Document the command in CLAUDE.md and README.

## 15. Magic link login with allowlist

Goal: The owner can log in via an emailed link; nobody else can (SPEC.md F1).

Description: Integrate the auth library from the auth ADR with the email provider from the email ADR, using a fake sender in tests. Only the allowlisted email receives a link; any other address sees the same neutral message and no email is sent. Links expire after 15 minutes and work once. Add tests for the three F1 acceptance criteria and tick them in SPEC.md.

## 16. Route protection and health check

Goal: Unauthenticated requests to app pages redirect to login.

Description: Add middleware or layout-level guards so every page requires a session except login, the token capture page (built in Phase 2) and a health check endpoint. Spec gap: the health check path and response are not defined; propose one and confirm. Test the redirect and each exclusion.

## 17. Entries service

Goal: User-scoped create, get, list, update and delete for entries (SPEC.md F2).

Description: Implement in `src/server/services` with Zod input schemas. Rules: free text is required, the date defaults to today, `raw_text` is never updated, status is `draft` or `confirmed`, every query is filtered by `user_id`, and hard delete also removes `entry_skills` links. Unit tests cover empty-text rejection and `raw_text` immutability.

## 18. Entry creation form and list page

Goal: The owner can add an entry and see entries newest first.

Description: Build the new-entry form (text, date, optional category, and the hint "Avoid confidential employer information.") and the reverse-chronological list page. Server actions or route handlers call the entries service only. Inputs are labelled, keyboard navigable and usable on a phone (SPEC.md §8).

## 19. Entry list filters in the URL

Goal: Filter by category, skill, status and date range, combined and reflected in the URL.

Description: Add filter controls to the list page whose state lives in the query string, so filters are shareable and back-button safe. Extend the entries service list function with the filter parameters and test combinations. Tick the F2 filter criterion in SPEC.md.

## 20. Entry detail, manual structuring and delete

Goal: Structured fields and date can be edited; delete asks for confirmation.

Description: Build the entry page showing `raw_text` read-only beside editable structured fields (`title`, `category`, `impact_type`, `impact_description`, `metric`, `feedback_from`, the four STAR fields) and the date. Saving structured fields can set the status to `confirmed`. Add delete with a confirmation step. Tests confirm `raw_text` is unchanged after edits.

## 21. Skills service

Goal: Per-user skill vocabulary with rename, merge and delete-if-unused (SPEC.md F5).

Description: Implement create (rejected with a pointer to the existing skill if the normalised name exists), rename (re-normalise and re-check), merge A into B (move entry links without duplicates, then delete A), delete (only if unused), and link and unlink of entry to skill. Uses `normalizeSkillName` from `src/domain`. Tests cover both F5 acceptance criteria; tick them.

## 22. Skills manage page

Goal: The owner can list, rename, merge and delete skills in the UI.

Description: Build the skills page over the skills service: list with usage counts, inline rename, a merge dialog that picks a target skill, and delete disabled while a skill is in use. Show the "already exists" pointer when a duplicate name is entered.

## 23. Link skills to an entry

Goal: On the entry page, pick existing skills or add a new one.

Description: Add a skill selector to the entry detail page that searches the user's vocabulary and allows adding a new skill through the skills service, so normalisation applies. Show linked skills as removable chips. Test that adding "power-bi" when "Power BI" exists links the existing skill instead of creating one.

## 24. Settings page without prompt fields

Goal: Edit display name, default output language and AI on/off (SPEC.md F9).

Description: Build a settings page and a user settings service backed by the `users` table. Only the fields above for now; prompt day, time, timezone and pause are added in Phase 2. Validate input with Zod and test the service.

## 25. Delete my account

Goal: One action removes all of the user's data (SPEC.md §8, Privacy).

Description: Add a "delete my account" action in settings, with confirmation, that deletes the user and all owned rows in every table and then ends the session. Test that no rows remain for that `user_id` in any table.

## 26. First end-to-end test

Goal: One browser test covers login, creating an entry and seeing it in the list.

Description: Install the tool from the e2e ADR, configure it against a local dev server and a test database, and obtain the magic link through a fake email sender or a test-only hook. Add the `e2e` command to CLAUDE.md and README and run it in CI, or document why it cannot run there yet.

---

# Phase 2 — Weekly prompt

## 27. ISO week and prompt due-time logic

Goal: Pure functions that compute a user's current ISO week and whether their prompt is due.

Description: In `src/domain`, implement a function returning the ISO week (for example `2026-W38`) for a date in a given timezone, and one that says whether the configured local day and time has already passed in the current ISO week. Test year boundaries (week 1 and week 53), DST transitions and different timezones. No database dependency.

## 28. Capture token handling

Goal: Pure, tested token generation and hashing (SPEC.md F3 Capture link, §8 Security).

Description: In `src/domain`, implement generating a random token of at least 32 bytes, encoding it for URLs, hashing it for storage, and constant-time comparison of a presented token against a stored hash. Tests cover length, uniqueness and that the stored value is never the raw token.

## 29. Email module with fake sender

Goal: One email sender interface with a fake for tests and a real adapter.

Description: In `src/server/email`, define a sender interface, a fake sender that records messages, and an adapter for the provider chosen in the email ADR, configured via env. Add the weekly prompt template containing the three questions from SPEC.md F3 and the capture link. Tests use the fake only; add the provider variables to `.env.example`.

## 30. Weekly prompt job

Goal: A secret-protected job endpoint that sends at most one prompt per user per ISO week.

Description: Implement a service that, for each user with prompts not paused, checks whether the prompt is due, upserts the `weekly_prompts` row for that ISO week (unique on user and week), creates a token and stores only its hash with an 8-day expiry, sends the email and records `sent_at`, `send_attempts` and `last_error`. Failed sends are retried on the next run within the same week. Expose it as `/api/jobs/weekly-prompt` requiring a secret. Tests cover the F3 scheduling criteria: idempotent re-run, mid-week settings change, paused users and retry.

## 31. Scheduler wiring

Goal: The job runs at least daily in production.

Description: Configure the scheduler chosen in the scheduler ADR to call the job endpoint daily with the secret. Commit any config file and document it in README. Verify with a manual trigger and record in the PR what you ran and saw.

## 32. Capture page

Goal: A token link opens a mobile-friendly form that creates draft entries for that week.

Description: Build the capture route: look up the token hash, reject expired or unknown tokens with a clear message and a login link, otherwise show three optional text fields. On submit, each non-empty answer becomes a `draft` entry with `source = weekly_prompt`, the `weekly_prompt_id`, `occurred_on` set to the last day of that ISO week, and a category hint of `learning` for question 2, `feedback` for question 3 and none for question 1. The token grants no read access to anything. Tests cover the F3 capture criteria.

## 33. Prompt settings

Goal: Prompt day, time, timezone and pause are editable in settings (SPEC.md F9).

Description: Extend the settings page and service with `prompt_day` (0–6), `prompt_time` (HH:MM), `timezone` (validated IANA name) and `prompts_paused`. Defaults are Friday 15:00 in `Europe/Berlin`. Tests validate input and, using the job service, confirm that changing settings mid-week does not produce a second prompt.

## 34. Real email check

Goal: One real weekly prompt email received by the owner (Phase 2 exit criterion).

Description: With real provider credentials in a local or staging environment, trigger the job manually, confirm the email arrives and open the capture link on a phone. Fix any formatting issues found and record the outcome in the PR. No new tests expected.

---

# Phase 3 — AI polishing

## 35. ADR: LLM provider and model

Goal: Decide the real LLM provider, model and data residency (SPEC.md Q1, D9).

Description: Compare providers on EU data residency, support for schema-constrained output, cost and SDK maturity, verifying in current official docs. Write the ADR including the model name and how the API key is configured. No code.

## 36. LlmProvider interface, MockProvider and prompt registry

Goal: The `src/ai` module skeleton from SPEC.md §7.1.

Description: Define the `LlmProvider` interface exactly as in §7.1, a `MockProvider` returning fixtures keyed by prompt id, and a prompt registry where each template has an `id`, a `version` and builders for the system and user text. Add a factory that returns the mock when no real provider is configured. Tests cover the mock and the registry; no provider SDK yet.

## 37. PolishProposal schema

Goal: A Zod schema for `PolishProposal` (SPEC.md §7.2) and its derived JSON schema.

Description: Implement the schema in `src/ai/schemas` using enums from `src/domain`: title up to 80 characters, up to 8 skills, impact description up to 300 characters, up to 3 `missing_info` questions, nullable fields as specified. Derive the JSON schema passed to providers from the same Zod schema. Tests: a category outside the enum is rejected (F4 criterion) and boundary lengths behave.

## 38. Polish prompt template v1

Goal: The prompt that turns `raw_text` into a `PolishProposal`.

Description: Write the system and user templates in `src/ai/prompts` with id `polish` and version 1. Rules to enforce: never invent facts, numbers or outcomes (unknown becomes `null` plus a question in `missing_info`); reuse the provided skill names exactly, otherwise return `is_new: true`; `feedback_from` is a role, not a name. Inputs are limited to `raw_text`, category hint, entry date and existing skill names. Add a matching mock fixture and a snapshot test of the rendered prompt.

## 39. Polish service

Goal: Create, validate and store an AI proposal for a draft entry (SPEC.md F4).

Description: Implement a service that builds the input (data minimisation per §7.5), calls the provider, validates with the `PolishProposal` schema, retries once on invalid output, and stores an `ai_proposals` row with status, output, error, provider, model, prompt id and version. Map proposed skills to existing ones by normalised name. Tests with the mock: valid output, invalid then valid, invalid twice becomes `failed`, skill mapping, and stored proposals record provider, model and version.

## 40. Accept, edit and reject proposals

Goal: Applying a proposal updates structured fields and status; rejecting leaves the draft intact.

Description: Extend the polish service with accept (copy fields to the entry, set `confirmed`, link skills, mark the proposal `accepted`), accept with edits (`edited`) and reject (`rejected`, entry stays `draft`, `raw_text` unchanged). New skills are created only when the user confirms each one. Tests cover the F4 rejection criterion and new-skill confirmation.

## 41. Proposal review UI and Polish button

Goal: The owner can review proposals side by side with the raw text.

Description: Add a "Polish" button on draft entries, hidden when AI is disabled or no provider is configured, and a review view with the raw text on one side and editable proposed fields on the other, per-skill confirmation for new skills, and accept, edit-and-accept and reject actions. Show the `missing_info` questions. For `failed` proposals, point the user to manual structuring.

## 42. Real provider adapter

Goal: The provider from the LLM ADR works behind `LlmProvider`.

Description: Implement the adapter in `src/ai/providers`, the only place the SDK is imported, using schema-constrained output where available while still validating in the app. Configure via env with new `.env.example` entries; the factory selects it only when configured and the user has AI enabled. Unit tests keep using the mock; describe one manually verified real call in the PR.

## 43. Polishing after weekly capture

Goal: New drafts from the capture page are polished automatically when AI is enabled.

Description: After a capture submission, run the polish service for each new draft when the user has AI enabled and a provider is configured. Spec gap: "queued" is not defined (in-request, background task or a queue); propose the simplest reliable option and confirm before building. Test with the mock that two answers produce two proposals and that nothing runs with AI disabled.

## 44. Privacy-safe logging

Goal: Logs contain IDs and error types only, never entry text or prompts (SPEC.md §7.5).

Description: Add a small logging helper used by services, the job and the AI module, with a test or lint guard against logging `raw_text`, prompt bodies or email addresses. Review existing log calls and replace them. Document the convention in CLAUDE.md.

---

# Phase 4 — Review prep

## 45. Generated outputs service

Goal: Store, list, view and delete generated outputs with source snapshots (SPEC.md F8).

Description: Implement a service over `generated_outputs` and `generated_output_entries` that saves type, parameters, content, language, provider, model, prompt id and version, and per source entry its id and a title snapshot. Regenerating always inserts a new row. When an entry is deleted, set `entry_id` to null but keep the snapshot so the UI can show the source as deleted. Tests cover the F8 criterion and the F2 delete criterion.

## 46. STAR story schema, prompt and service

Goal: Generate a STAR story or `needs_input` questions for a confirmed entry (SPEC.md F6).

Description: Add the `StarResult` Zod schema (§7.3, at most 3 questions), a `star_story` prompt template version 1 with language (EN/DE), length (short or standard) and a no-invented-facts rule, and a service that calls the provider, validates, and saves an `ok` result through the generated outputs service. Tests with the mock: a missing Result yields `needs_input`; the saved output records source entry, parameters, provider, model and prompt version.

## 47. STAR story UI

Goal: "Generate STAR story" on a confirmed entry, with a questions round trip.

Description: Add the button with language and length options. When the result is `needs_input`, show the questions with answer fields; after the user confirms, save the answers to the entry's STAR fields and generate again. Show the story with a link to the saved output. Hidden when AI is disabled.

## 48. Period summary schema, prompt and service

Goal: Generate a cited `PeriodSummary` from confirmed entries in a date range (SPEC.md F7).

Description: Add the `PeriodSummary` Zod schema (§7.4, each bullet cites at least one entry id) plus a validator that rejects bullets citing ids outside the input set, and a `period_summary` prompt template version 1 taking purpose and language. The service selects confirmed entries by date range and optional category and skill filters, returns an explicit empty-selection result without calling the provider, and stops with a clear error when the input exceeds a context budget. Spec gap: the budget size is not defined; ask. Tests with the mock cover all three F7 criteria.

## 49. Period summary UI

Goal: Pick a range, filters, purpose and language; read bullets linked to their entries.

Description: Build the page with presets (last 3, 6 or 12 months) or a custom range, category and skill filters, and purpose and language selectors. Render sections and bullets with each bullet linking to its source entries, plus "copy as text" and "download as Markdown". Show the empty-selection and range-too-large messages.

## 50. Saved outputs UI

Goal: List, view and delete saved outputs; older versions stay viewable.

Description: Build the outputs list (type, date, parameter summary) and a detail view that renders STAR stories and period summaries, including markers for deleted source entries. Add delete with confirmation and a "regenerate" action that creates a new output and links back to the previous one.

---

# Phase 5 — Ship

## 51. Deploy

Goal: The app and database run on the hosting chosen in the hosting ADR.

Description: Create the production environment, set every variable from `.env.example`, run migrations, enable the scheduler, and confirm the health check, login and one weekly prompt work in production. Document the deploy steps and rollback in README.

## 52. Export as JSON and Markdown (F10, Could)

Goal: Download all entries and skills as JSON, and entries as Markdown.

Description: Add an export service producing a JSON document of entries (with linked skill names) and skills, and a Markdown document of entries grouped by month. Expose both as downloads from settings. Tests check that the JSON round-trips the data and includes only the current user's rows.

## 53. Accessibility and responsive review

Goal: The capture page and entry form meet SPEC.md §8 on a phone and with a keyboard.

Description: Audit all pages for semantic HTML, labelled inputs, keyboard navigation and visible focus, and test the capture page and entry form at phone width. Fix what you find and add e2e checks for keyboard-only entry creation and the mobile viewport.

## 54. README, screenshots and ADR tidy-up

Goal: A portfolio-ready README and consistent ADRs.

Description: Write the README with purpose, screenshots, the architecture diagram from SPEC.md §10, setup, commands and links to the ADRs. Review every ADR for a consistent format and current status, and close out the open questions table in SPEC.md §12 with the decisions taken.
