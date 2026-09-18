# Work Logbook — Specification v1

| | |
|---|---|
| Status | Draft 0.1 |
| Owner | Kitty |
| Last updated | 2026-09-18 |
| Source of truth | This file. Changes to scope or behaviour are made here first. |

---

## 1. Purpose

A personal web app to capture achievements with minimal effort and turn them into reusable material for appraisals, interviews and CVs.

- **Capture:** a weekly email prompt links to a short form; entries can also be added any time.
- **Structure:** AI turns free text into a fixed, consistent structure that the user confirms.
- **Reuse:** on demand, AI generates STAR stories and period summaries from confirmed entries.

**Goals of the build itself**
1. Practise agentic, spec-driven development with Claude Code.
2. Produce a clean, portfolio-grade repository (readable code, tests, ADRs, README).

---

## 2. Users

- **v1:** single user (the owner).
- **Designed for multi-user later:** every user-owned record carries `user_id`, every query is scoped by it, and there are no global singletons for user data. Sign-up stays disabled in v1.

---

## 3. Scope

### In scope (v1)

| ID | Feature | Priority |
|---|---|---|
| F1 | Authentication (owner only) | Must |
| F2 | Entries: create, list, filter, edit, delete | Must |
| F3 | Weekly prompt email with token-based capture page | Must |
| F4 | AI polishing: free text → fixed structure, user confirms | Must |
| F5 | Skills vocabulary: consistent names, rename, merge | Must |
| F6 | STAR story generation, on demand | Must |
| F7 | Period summary generation, on demand | Must |
| F8 | Saved generated outputs | Must |
| F9 | Settings | Must |
| F10 | Data export (JSON + Markdown) | Could |

### Out of scope (v1)

- Native mobile app (responsive web only)
- Integrations: Outlook, Teams, GitHub, LinkedIn import or posting
- Sharing, public profiles
- Pattern spotting across time (→ v2)
- Web push / PWA notifications (→ v2)
- Reply-by-email capture (→ v2)
- Multi-user sign-up (→ later)

---

## 4. Decision log (scoping Q&A, 2026-09-18)

| # | Topic | Decision | Rationale |
|---|---|---|---|
| D1 | Users | Single user, multi-user-ready data model | Keep v1 simple without blocking growth |
| D2 | Entry types | Work achievements, learning, side projects, feedback received | Cover job and personal development |
| D3 | Prompt channel | Weekly email with link to a pre-filled form | Simplest, works on any device, no app store |
| D4 | Later channel | PWA web push (v2) | Possible without app store |
| D5 | AI feature order | Polishing → review/interview prep → pattern spotting | Polishing improves data quality that later features depend on; patterns need months of data |
| D6 | Entry shape | Free text at capture; AI proposes fields in a **fixed schema**; user accepts or edits | Low capture friction plus consistent structure |
| D7 | Consistency | Closed enums for category and impact type; skills must reuse the existing vocabulary; new skills need confirmation; every AI response is validated against the schema | Prevents drift like "Power BI" vs "PowerBI" |
| D8 | Review prep | Both STAR stories (per entry) and period summaries — only generated when the user asks | User stays in control; no background AI calls for outputs |
| D9 | AI provider | Deferred; provider hidden behind an interface; app must work without AI | Keep options open (incl. data residency) |
| D10 | Stack | TypeScript full-stack: Next.js + PostgreSQL | One language end to end, common for agentic coding, portfolio value |
| D11 | Hosting | Managed PaaS + managed Postgres, free/cheap tier | Low cost, low ops; concrete choice in Phase 0 |

---

## 5. Functional requirements

### F1 — Authentication

- Login by email magic link.
- Only the email address in the allowlist (env var) can log in. Sign-up is disabled.
- Session required for all pages except: login, token capture page (F3), health check.

**Acceptance criteria**
- [ ] An unauthenticated request to an app page redirects to login.
- [ ] A login attempt with a non-allowlisted email sends no email and shows a neutral message (no account enumeration).
- [ ] Login links expire after 15 minutes and work once.

### F2 — Entries

- Create an entry with **free text (required)** and a date (default: today). Category is optional at creation.
- The original text is stored as `raw_text` and never overwritten. Structured fields are stored separately.
- Entry status: `draft` (not yet structured) or `confirmed` (structured fields accepted).
- List view: reverse chronological; filter by category, skill, status and date range.
- Edit structured fields and date; delete with confirmation (hard delete).
- The entry form shows a hint: *"Avoid confidential employer information."*

**Acceptance criteria**
- [ ] Creating an entry with empty text is rejected.
- [ ] Editing structured fields never changes `raw_text`.
- [ ] Filters can be combined and are reflected in the URL (shareable, back-button safe).
- [ ] Deleting an entry also removes its skill links; saved outputs that referenced it keep their text but show the source as deleted.

### F3 — Weekly prompt and capture

**Scheduling**
- A scheduled job runs at least daily (UTC). For each user with prompts enabled, it sends the weekly prompt once the user's configured local day and time has passed in the current ISO week.
- At most one prompt per user per ISO week (idempotent; safe to re-run).
- Defaults: Friday 15:00, `Europe/Berlin` (provisional, see §12). Configurable in settings.
- The job endpoint is protected by a secret.

**Email**
- Three optional questions and one link to the capture page:
  1. *What did you achieve or move forward this week — at work or in side projects?*
  2. *What did you learn?*
  3. *Did you receive any feedback?*

**Capture link**
- Contains a random token (≥ 32 bytes). Only its hash is stored.
- Valid for 8 days and can be used multiple times until it expires.
- Scope: create entries for that prompt's week only. It grants no access to view or edit any other data.

**Capture page**
- Three text fields (one per question), all optional; mobile-friendly.
- On submit, each non-empty answer becomes one `draft` entry with `source = weekly_prompt`, `occurred_on` = last day of that week, and a category hint: Q2 → `learning`, Q3 → `feedback`, Q1 → none (AI proposes).
- If AI is enabled, polishing (F4) is queued for each new draft. Reviewing proposals happens in the logged-in app.

**Acceptance criteria**
- [ ] Running the job twice in the same week sends only one email.
- [ ] Changing the timezone or day setting mid-week does not produce a second email that week.
- [ ] An expired or unknown token shows a clear message and a link to log in.
- [ ] A valid token cannot read any existing entry.
- [ ] Submitting two non-empty answers creates exactly two draft entries.
- [ ] Prompts can be paused in settings; paused users receive nothing.
- [ ] A failed email send is logged and retried on the next job run within the same week.

### F4 — AI polishing

- Runs automatically after weekly capture and on demand via a "Polish" button on any draft entry.
- **Input to the AI:** `raw_text`, category hint, entry date, the user's existing skill names. Nothing else — no email, no name.
- **Output:** a `PolishProposal` (§7.2), validated against the schema.
  - Invalid output → retry once → still invalid: proposal marked `failed`; the user can structure the entry manually.
- **Rules the prompt must enforce**
  - Do not invent facts, numbers or outcomes. If the impact or a metric isn't in the text, set it to `null` and add a question to `missing_info`.
  - Reuse existing skill names exactly. Any other skill is returned with `is_new: true`.
- **Review UI:** raw text next to proposed fields; the user can accept all, edit and accept, or reject. Accepting sets the entry to `confirmed`. New skills are added to the vocabulary only after the user confirms each one.
- **Without a configured AI provider** the app is fully usable: polish buttons are hidden and entries are structured manually.

**Acceptance criteria**
- [ ] A proposal with a category outside the enum is rejected by validation.
- [ ] A proposal skill that matches an existing skill after normalisation (F5) is mapped to that skill, not created again.
- [ ] Rejecting a proposal leaves the entry as `draft` with `raw_text` unchanged.
- [ ] Every stored proposal records provider, model and prompt version.
- [ ] Tests run against a mock provider only.

### F5 — Skills vocabulary

- Per-user list of skills; entries link to skills (many-to-many).
- **Normalisation** for uniqueness: lowercase, then remove whitespace, `-`, `_` and `.`. Example: "Power BI", "PowerBI" and "power-bi" all become `powerbi`. The display name keeps the user's spelling.
- Manage page: rename, merge (moves all entry links to the target skill), delete (only if unused).

**Acceptance criteria**
- [ ] Creating a skill whose normalised name already exists is rejected with a pointer to the existing skill.
- [ ] Merging A into B moves all of A's entry links to B and deletes A, without duplicate links.

### F6 — STAR story (on demand)

- "Generate STAR story" button on a confirmed entry. Options: language (EN/DE), length (short ≈ 80 words / standard ≈ 200 words).
- If Situation, Task, Action or Result can't be stated without inventing information, the AI returns `needs_input` with up to 3 questions instead of a story (§7.3). The user answers; the answers are saved to the entry's STAR fields after the user confirms, then the story is generated.
- The result is saved as a generated output (F8) linked to the entry.

**Acceptance criteria**
- [ ] With a missing Result, the response is `needs_input`, not a story.
- [ ] The saved output records its source entry, parameters, provider, model and prompt version.

### F7 — Period summary (on demand)

- **Inputs:** date range (presets: last 3, 6 or 12 months; or custom), optional filters (categories, skills), purpose (`appraisal` | `interview` | `cv`), language (EN/DE).
- Uses **confirmed entries only.**
- **Output:** a `PeriodSummary` (§7.4): sections with bullets. Every bullet cites at least one source entry ID.
- Validation rejects bullets that cite IDs that were not in the input set.
- The UI shows each bullet with links to its source entries. Actions: copy as text, download as Markdown.
- If the selected entries exceed the context budget, generation stops with a clear message asking the user to narrow the range. Chunking is out of scope for v1.

**Acceptance criteria**
- [ ] A bullet citing an entry outside the selection fails validation.
- [ ] An empty selection shows a message and makes no AI call.
- [ ] Each bullet links to its source entries.

### F8 — Saved outputs

- List, view, delete generated outputs.
- Regenerating creates a new output; the old one is kept.
- Stored per output: type, parameters, content, source entry IDs, provider, model, prompt version, timestamp.

**Acceptance criteria**
- [ ] Regenerating an output leaves the previous version intact and viewable.

### F9 — Settings

- Prompt day and time, timezone, pause prompts, default output language, AI on/off.

### F10 — Export (Could)

- Download all entries and skills as JSON, and entries as Markdown.

---

## 6. Data model

All user-owned tables have `user_id` (FK → `users.id`), `created_at`, `updated_at`. IDs are UUIDs.

**users**
- `id`, `email` (unique), `display_name`, `timezone` (default `Europe/Berlin`), `prompt_day` (0–6), `prompt_time` (HH:MM), `prompts_paused` (bool), `default_language` (`en` | `de`), `ai_enabled` (bool)

**entries**
- `id`, `user_id`, `raw_text` (immutable), `occurred_on` (date), `status` (`draft` | `confirmed`), `source` (`manual` | `weekly_prompt`), `weekly_prompt_id` (nullable)
- Structured fields (nullable until confirmed): `title`, `category`, `impact_type`, `impact_description`, `metric`, `feedback_from` (role, not a name — e.g. "manager"), `star_situation`, `star_task`, `star_action`, `star_result`

**skills**
- `id`, `user_id`, `name`, `normalized_name` — unique on (`user_id`, `normalized_name`)

**entry_skills**
- `entry_id`, `skill_id` — primary key on both

**weekly_prompts**
- `id`, `user_id`, `iso_week` (e.g. `2026-W38`), `sent_at`, `send_attempts`, `last_error`, `token_hash`, `expires_at`
- Unique on (`user_id`, `iso_week`)

**ai_proposals**
- `id`, `user_id`, `entry_id`, `status` (`pending` | `failed` | `accepted` | `edited` | `rejected`), `output_json`, `error`, `provider`, `model`, `prompt_id`, `prompt_version`

**generated_outputs**
- `id`, `user_id`, `type` (`star_story` | `period_summary`), `params_json`, `content_json`, `language`, `provider`, `model`, `prompt_id`, `prompt_version`

**generated_output_entries**
- `output_id`, `entry_id` (nullable after the entry is deleted), `entry_title_snapshot`

**Enums (defined once in code, reused by DB, validation and UI)**
- `category`: `work` | `learning` | `side_project` | `feedback`
- `impact_type`: `time_saved` | `quality` | `cost` | `risk_reduction` | `visibility` | `learning` | `other`
- `purpose`: `appraisal` | `interview` | `cv`

---

## 7. AI contract

### 7.1 Provider interface

All AI calls go through one internal interface. No provider SDK is imported outside the AI module.

```ts
interface LlmProvider {
  readonly name: string;   // e.g. "mock"
  readonly model: string;
  generateStructured(args: {
    promptId: string;
    promptVersion: number;
    system: string;
    user: string;
    jsonSchema: object;    // derived from the same schema used for validation
    maxOutputTokens: number;
  }): Promise<unknown>;    // caller validates the result
}
```

- `MockProvider`: deterministic fixtures, used in tests and local development.
- Real provider: chosen via an ADR before Phase 3. If the provider supports schema-constrained output, use it — but always validate in the app regardless.
- Prompt templates live in the AI module, each with an `id` and a `version`. Any change to a prompt bumps its version.

### 7.2 PolishProposal

```ts
type PolishProposal = {
  title: string;                         // max 80 chars
  category: Category;
  skills: { name: string; is_new: boolean }[];   // max 8
  impact_type: ImpactType | null;
  impact_description: string | null;     // max 300 chars
  metric: string | null;                 // only if stated in raw_text
  feedback_from: string | null;          // role only, category "feedback"
  star: {
    situation: string | null;
    task: string | null;
    action: string | null;
    result: string | null;
  };
  missing_info: string[];                // questions for the user, max 3
};
```

### 7.3 StarResult

```ts
type StarResult =
  | { status: "ok"; story: string }
  | { status: "needs_input"; questions: string[] };   // max 3
```

### 7.4 PeriodSummary

```ts
type PeriodSummary = {
  title: string;
  sections: {
    heading: string;
    bullets: { text: string; entry_ids: string[] }[];   // entry_ids: min 1
  }[];
};
```

### 7.5 Data minimisation

- Send only the fields listed per feature. Never send the user's email or name.
- Don't write entry text or prompts to production logs; log IDs and error types only.

---

## 8. Non-functional requirements

- **Security:** random tokens (≥ 32 bytes), store hashes only; secret-protected job endpoint; framework defaults for CSRF and cookies; secrets only in environment variables; `.env.example` kept current.
- **Privacy:** prefer EU region for the database; "delete my account" removes all user data; no analytics or third-party trackers.
- **Reliability:** scheduled job is idempotent; failed sends retried on the next run.
- **Accessibility:** semantic HTML, labelled inputs, keyboard navigable, visible focus.
- **Responsive:** capture page and entry form fully usable on a phone.
- **Language:** entries can be in any language; generated outputs in EN or DE as selected. UI language: see §12.
- **Quality:** CI runs lint, typecheck and tests on every push.

---

## 9. Tech stack

**Decided**
- TypeScript (strict mode)
- Next.js (App Router), server-side logic in a separate service layer
- PostgreSQL
- Zod for all validation (inputs, env vars, AI output)
- GitHub + GitHub Actions for CI

**To decide in Phase 0** — one ADR each; verify current versions, APIs and pricing in official docs before deciding:
- ORM and migrations
- Auth library (magic link)
- Transactional email provider
- Hosting + managed Postgres (EU region, free/cheap tier)
- Scheduler mechanism (platform cron vs external)
- Unit and end-to-end test tools

**To decide before Phase 3**
- LLM provider and model (incl. data residency)

---

## 10. Architecture

```
Browser
  │
Next.js (pages, forms, route handlers)
  │
Service layer (src/server/services)  ── AI module (src/ai) ── LlmProvider (mock | real)
  │                                  └─ Email module (src/server/email)
DB layer (src/db) ── PostgreSQL

Scheduler ──(secret)──▶ /api/jobs/weekly-prompt
```

**Proposed layout**

```
src/
  app/               # routes and UI
  components/
  domain/            # enums, types, pure logic (e.g. skill normalisation, ISO week)
  server/
    services/        # business logic, always user-scoped
    email/
  ai/
    providers/       # mock + real
    prompts/         # versioned templates
    schemas/         # Zod schemas for AI I/O
  db/                # schema, migrations, queries
tests/
docs/
  adr/
SPEC.md
CLAUDE.md
README.md
```

---

## 11. Milestones

| Phase | Content | Exit criteria |
|---|---|---|
| 0 — Setup | ADRs for §9 open items; scaffold; lint, typecheck, test; CI; `.env.example`; README skeleton | CI green on an empty app; ADRs merged |
| 1 — Core | Data model and migrations; F1 auth; F2 entries (manual structuring); F5 skills; F9 settings (without prompt fields) | F1, F2, F5 acceptance criteria pass |
| 2 — Weekly prompt | F3 scheduler, email, token capture; prompt settings | F3 acceptance criteria pass; one real email received |
| 3 — AI polishing | Provider ADR; interface + mock; real provider; F4 | F4 acceptance criteria pass |
| 4 — Review prep | F6, F7, F8 | F6–F8 acceptance criteria pass |
| 5 — Ship | Deploy; F10 (if time); README with screenshots; tidy ADRs | App live; owner uses it for 4 weeks |

---

## 12. Open questions

| # | Question | Provisional answer | Needed by |
|---|---|---|---|
| Q1 | LLM provider and data residency | — | Phase 3 |
| Q2 | UI language: English or German? | English | Phase 1 |
| Q3 | Default prompt day and time | Friday 15:00, Europe/Berlin | Phase 2 |
| Q4 | Repo public or private? If public: no personal data in seeds, fixtures or commits | Private until Phase 5 | Phase 0 |
| Q5 | Hard delete vs soft delete for entries | Hard delete | Phase 1 |

---

## 13. v2 backlog

- Pattern spotting (themes, skill growth, gaps over time)
- PWA install + web push reminders
- Reply-by-email capture
- Formatted CV / LinkedIn export
- Multi-user sign-up
- Integrations
