Additions to this document must be minimal and concise. Embolding and all caps are reserved for negation of clauses, not used to emphasize points. Narration on why these instructions exist should not be present in this document. Skill details should live in the skill files, not this document.

##What we are building:

[REDACTED: product/business description]

Four frontend apps plus one shared backend API and database. Each app is a
separate portal over the same data model; some features are gated per app.

All subprojects live in a single private GitHub repo (`ORG/REPO`, `main`
branch). The old per-subproject repos are deprecated.


Subproject default paths:
- `app_a_njs/` — Next.js, port 3001
- `app_b_njs/` — Next.js, port 3003
- `app_c_njs/` — Next.js, port 3002
- `app_d_njs/` — Next.js, port 3004
- `api_fastapi/` — FastAPI backend, port 8000

The Next.js servers hot reload. FastAPI only reloads if it was started with `--reload` — a
long-running `uvicorn app.main:app --port 8000` serves whatever the code looked like when it
started, silently. That has already produced a false "the fix didn't work" reading during
verification. Before trusting anything you measure against port 8000, check the process:
`ps -eo pid,lstart,command | grep uvicorn`. If it predates your edits or lacks `--reload`, start
your own on a spare port instead.

All repos auto deploy on pushes to main git branch (ignored on no diff).

Note: the prod FastAPI API runs on Render; the prod DB is AWS RDS (private, SSM-only).

### Staging environment (full cloud replica, safe to test against)
There is a complete cloud staging stack (Render FastAPI + Supabase DB + Vercel previews) that mirrors prod but is fully isolated from AWS prod data — safe for prod-like end-to-end testing including real mutations. URLs, the persistent `staging` branch (**never** commit directly to it), and refreshing it (`scripts/refresh-staging.sh`): use the **`staging`** skill.


### Prod safety (applies to this entire file)
Ask the user before any prod mutation — `migrate_prod.sh up`, prod DB value changes, `tf.sh apply`.
Read-only prod queries are fine and need no confirmation.
Exception: "ship it" authorizes the prod mutations the shipped work needs — apply them without
re-asking per file. That covers *your* work only: never sweep in another session's pending
migration, and `tf.sh apply` still needs its own OK.

### Planning, Implementation, and Delegation Notes

#### Scale rigor to blast radius (governs everything below)
Pick the path from what the change can break:

- Light path — read-only work, local-only changes, docs, copy tweaks, single-file edits, and
  anything trivially reversible: make the change, `npm run build` (or compile) if code, done. No
  reference sweep, no agent, no real-flow drive, no independent verification pass.
- Full protocol — prod mutations, migrations, schema changes, auth, payments, [regulated data], compliance
  surfaces ([regulated workflows]), cross-app contracts, and anything touching shared
  state or another session's work: real-flow verification, full reference sweep per the brownfield
  note below, and an independent check of the merged result.

When unsure which path applies, say which one you're taking and why in one line, then proceed —
don't ask. Escalate mid-task if the change turns out to touch the full-protocol list.

#### Planning
- Create clear plans in /implementation_plans with specific file lists, endpoint specs, and schemas to help with coordination among agents.  Always give agents precise instructions and context.
- Avoid using Plan Mode or Plan Agent as it disrupts workflows - prefer writing plans to /implementation_plans instead without entering Plan Mode or using Plan Agent
- When planning refactors or brownfield changes, think about potential unintended effects of the changes, grep/search for all references to the legacy field/function/API (including comments, tests, configs, and indirect calls), and plan accordingly.

#### Talking to the user
- Please ask questions in prose rather than using AskUserQuestion
- Whenever presenting options to the user, if the quickest fix/implementation and the most architecturally sound fix/implementation differ, please indicate as much.

#### Implementation
- LLM inference is costly and can be lossy.  When attempting to copy large content, code, or files, prefer using system commands like cp or sed or scripts rather than regurgitating code with your own inference.
- Screenshot every new component and every modal/dialog you touched, at the width it ships at,
  before reporting it done. `innerText` and the a11y snapshot report a component that has blown its
  container or pushed content off-screen as correct.

#### Delegation
- Once plan is created or change is decided, please delegate implementation to (an) agent(s) running in the background unless told otherwise.  However, if you're already deep in the code, it may be better for you to implement without delegation. 
- The Explore agent tends to over explore.  Give it clear return items and tell it to explore the minimum possible to answer the questions.  You can always call it again for more info.  It also tends to be overconfident, so watch for and interrogate ungrounded claims.
- When polling background agents, don't grep for a specific completion string — just `tail` the output file to see the latest activity. This avoids false "still running" results from mismatched grep patterns and keeps context small.

#### Keeping subagents in scope
Subagents execute a brief exhaustively and literally. Constrain what they change; leave what they
investigate thorough.

- The brief's length affects the output's length. An eight-item numbered checklist produces an
  eight-item exhaustive pass. Ask for the smallest thing that answers the question — "screenshot
  the sign-off modal and the queue" rather than "review the UI across these 8 surfaces at 2 widths".
- Scope by mechanism. Don't ask whether a fix was in the brief; ask what making it takes. Tell
  subagents:
  - Same defect, same mechanism as the assigned fix → just fix it, no permission needed. If the
    brief says paginate one unbounded query, paginate every sibling that has the identical defect.
    Report the full list of sites touched.
  - Same defect but it needs a mechanism the brief didn't sanction (a changed helper signature, a
    different sort, an extra query) → fix it and flag the mechanism shift, so the lead reviews
    the new mechanism.
  - Different defect class, or anything a caller can observe — response shape, a documented
    ordering, schema, another app, shared state → **report, don't fix**. Also anything needing a
    prod mutation or a migration: hand it back.
  - Exception that overrides all of the above: if the assigned change can't be verified without the
    fix, make the fix.
- Before escalating on "this changes behavior", check whether the behavior was ever specified.
  "This would change the output ordering" is not a compatibility constraint if the ordering among
  tying rows was never defined.
- **No** incidental *destructive* shared-state mutation. Subagents should not modify or delete
  existing local DB rows, another portal's config, or admin settings unless the brief says to.
  Additive is fine — new rows, new fixtures, new records may be created without asking; report
  what was added. If verification requires changing something that already exists, create a scoped
  fixture, restore it afterwards, and report exactly what was touched.
- Stray-file gate before every commit: `git status --porcelain` first. **Don't** commit
  `package.json` / lockfile changes produced by running build tooling in a worktree — those are
  environment artifacts. Lead agents should check the subagent's branch before merging.
- Subagents are good at catching errors in your brief — the corrections are usually worth taking,
  but verify them; they also sometimes "correct" something that was already right.

#### If you are a subagent, read this
Subagents receive this entire file (verified 08/10/26). Most of it is written in the second person to the
main assistant, so resolve the ambiguous rules as follows — these override any conflicting
instruction in your brief:

- You have **no interface** to the user. Every rule here of the form "ask the user first" / "without
  asking the user" means: stop and report to the agent that launched you.
- **Never** push to `main`, **never** merge, **never** deploy. Commit on your own branch and report. The
  "Ship it" definition in this file (commit → push → merge → apply to prod) describes the main
  assistant's flow.
- **Never** mutate prod. No `migrate_prod.sh up`, no prod writes, no `tf.sh apply`. Read-only prod
  queries via `read_prod.sh` are fine. Hand the mutation back to your caller.
- **Don't** re-delegate your assignment. Playwright slots, alt ports, and worktrees are allocated by
  the agent that launched you; a nested agent has no allocation and will collide with another
  session's browser or servers. You may spawn a read-only Explore agent for a parallel search;
  don't spawn anything needing a browser, dev server, port, or worktree. If the work needs one,
  stop and ask your caller to allocate it. Report any agent you spawn and its token cost.
- The "Keeping subagents in scope" rules above are about you — apply them to yourself: scope by
  mechanism (fix same-mechanism siblings, flag mechanism shifts, report observable-contract
  changes), no incidental destructive shared-state mutation (additive is fine), stray-file gate
  before every commit.
- The blast-radius rule at the top applies to you too. If your brief's rigor looks mismatched to
  what the change can break, say so in your report.

### Shipping (commit → push → merge → prod)
- Use the **`shipping`** skill before any commit/push/merge/deploy — it holds the full flow (stray-file check, `npm run build` gate, deploy verification, shared-index rule, migration ship order).
- If the user just says "commit", that means commit only — **do not** push, merge, or run the rest of the shipping flow.
- Land branches on main via rebase/fast-forward rather than a merge commit, for readable history.
- A push is **not** a deploy. Vercel's Ignored Build Step can skip an app's build silently, leaving prod on its old bundle with the code on main. `scripts/verify-deploy.sh snapshot` before pushing, `wait` after.

### Testing Notes
- Playwright slots/servers, browser cleanup, locator timeouts, dev-mode shortcuts, and all test-account logins: use the **`testing`** skill.
- Often, curl + psql is a great prerequisite before Playwright testing.

### Searching
- **Search with `rg`, not `grep`.** There is no `Grep`/`Glob` tool on native macOS/Linux builds —
  ripgrep is embedded in the Claude Code binary and exposed as an `rg` shell function instead, so
  it's always available in Bash but never shows up in the tool list. Use it: `rg` respects
  `.gitignore`, so `node_modules`, `.next`, and `.git` are skipped without extra flags.
  If you must use `grep`, exclude at traversal time: `--exclude-dir=node_modules
  --exclude-dir=.next`. Piping to `| grep -v node_modules` does not work: `grep -r` has already
  walked every `node_modules` and read every matching file before the pipe filters the output.
  Worktrees carry their own cloned `node_modules`, so this costs more there.

### Keep the primary checkout on `main`
- Preferred: delegate to a background agent with `isolation: worktree` (Agent tool). If it needs dev servers or a browser, also claim a slot for that worktree — `scripts/agent-slot.sh attach <slot> <worktree-path> <app>` — which reserves its ports + Playwright instance and boots the servers. (`agent-slot.sh up <slot> <branch> <app>` does the same but creates the worktree itself; prefer `attach`, since `worktree-gc.sh` only ever cleans up `.claude/worktrees`.)
- Parallel agents (different terminals) should **not** share the `main` checkout — spin up isolated worktree + server stacks with `scripts/agent-slot.sh`: use the **`agent-slots`** skill.
- If working directly in a worktree, run that worktree's own dev servers on alt ports to verify — the user's servers on 3001/3002/8000 watch the main checkout and won't hot-reload worktree edits. **Don't improvise ports:** slots 1..7 reserve `8010-8070` and `3011-3074`, so a worktree without a slot uses 8080+ (backend) and 3080+ (frontends).
- Testing a Next.js frontend (or FastAPI backend) from a git worktree has env/tooling snags (hardcoded dev port, `node_modules` — clone it with `cp -Rc`, never symlink, or Turbopack panics and you end up building with a bundler Vercel doesn't use — missing `.env.local`): use the **`worktree-nextjs-testing`** skill.
- Commit/push/merge from the worktree, then tear it down (`agent-slot.sh down <slot>`). The primary checkout stays on `main` throughout.
- Carve-out: quick single-file edits and read-only tasks don't need a worktree — just work in place (no branch switch needed for read-only).

### Other notes
- You will often see uncommitted or committed changes from parallel sessions unrelated to your work. Don't modify or delete them without asking. If you're unsure what to do with the unrelated work — ask the user, or the agent who made the changes.
- Write new docs to `temp_docs/` unless told otherwise. `docs/` holds durable reference material —
  read from it freely, but only add there when asked.
- Please avoid modifying environment files

### Local Database Access

```bash
# Local Supabase (after `supabase start`)
PGPASSWORD=<REDACTED> psql -h 127.0.0.1 -p 6543 -U <REDACTED_USER> -d <REDACTED_DB>
```

### Local DB lifecycle (apply migrations locally, reset)
- Applying new local migrations and resetting local (test-account reseed via `db-reset-local.sh`): use the **`local-db`** skill.
- **Never** reset the local db without user confirmation.

### Schema Lookup
See `/docs/db/DATABASE_SCHEMA.md` for full table list and descriptions.
```bash
./scripts/db-schema.sh users              # Show table schema
./scripts/db-schema.sh --list             # List all tables
```

### Migrations
- Migration files live in the root `supabase/migrations`. Before writing, reviewing, or applying one, use the **`migrations`** skill (naming/ledger rules, no `BEGIN;`/`COMMIT;`, RLS + `SECURITY DEFINER` grant closure, AWS prod specifics).
- Apply schema changes (ALTER, CREATE TABLE, DROP, etc.) by writing the migration file first, then applying it with `psql -f <migration_file>` — not as direct psql statements. Read queries and data inspection via psql are fine.
- Ship order for prod migrations: see the **`shipping`** skill.

### Prod mutations: only through the vetted path
- Every prod DB mutation goes through `migrate_prod.sh up`. Even a one-row data fix should be a tiny migration, which keeps a single, reviewable entrypoint.
- Ownership test before putting DML in a migration. Ask who owns the value's current state. If an
  admin, client orgs, provider, or end users can change it in the UI, a migration must not restate it — the
  ledger is a permanent instruction to set it again. That includes derived values an override flag
  can pin (`*_is_manual` on fulfillment prices): recompute only where the flag is unset.
- DML that does belong in a migration must be scoped so a replay is a no-op. Key on the rows that
  existed — an explicit id/slug list, or `WHERE created_at < '<date>'` — never on a live state value
  like `WHERE status = 'pending'`. "Existing rows" in a comment is not a WHERE clause.
- `up` refuses a full replay against a populated database (`scripts/sb_migrate/replay_guard.sh`).
  A restored or rebuilt stack takes `baseline`, not `up`.

### Applying Migrations to Self-Hosted Prod (AWS RDS + Supabase Stack on AWS)
- Prod is SSM-only (private RDS) — apply migrations with the ledger-backed `migrate_prod.sh status` / `up`: use the **`prod-migrations`** skill. AWS-specific constraints (pg_cron limits, EventBridge crons, `tf.sh apply` verification) are in the **`migrations`** skill.

### Querying Prod Database
- Read live prod (SSM → `<REDACTED_DB_ROLE>`, [regulated data] tables DB-revoked) via `scripts/sb_migrate/read_prod.sh`: use the **`prod-db-read`** skill.


### Supabase Python Client: Avoid `maybe_single()`

Don't use `.maybe_single()` - it returns 204 "Missing response" errors instead of `None`. Use `.execute()` with manual checks:
```python
result = supabase.table("x").select("*").eq("col", val).execute()
row = result.data[0] if result.data else None
```

### Calling FastAPI Endpoints Locally (Auth)
- Get a Supabase JWT and call the local backend (`source scripts/api-token.sh`, then `$TOKEN`): use the **`fastapi-local-auth`** skill.

### Login info for Playwright testing
- Test accounts for every portal live in the **`testing`** skill.

### Date-Only Strings in TS: Use `formatDateOnly()` from `lib/date-utils`

**Never** use `new Date("YYYY-MM-DD")` directly — JS parses date-only strings as UTC, which shifts the date back one day in negative-offset timezones. Use `formatDateOnly()` or `parseLocalDate()` from `app_a_njs/lib/date-utils.ts` instead.

### PostgREST: explicit FK hint when multiple FKs exist between two tables

When two tables have more than one FK relationship (e.g. `users.org_id → organizations.id` plus `organizations.suspended_by → users.id`), PostgREST embeds like `.select("*, organization:organizations(*)")` fail because the join is ambiguous. Add the FK name: `.select("*, organization:organizations!users_org_id_fkey(*)")`. When adding FK columns between already-related tables, grep existing queries and add explicit hints.

### Python: avoid `value or default` for numeric/falsy fields

`value or default` silently replaces `0`, `""`, `[]`, and `False` with the default. Use `value if value is not None else default` when those are valid inputs.

### Never derive a count from a fetched page

A number shown to a user must come from a server-side count, never `.filter(...).length` over rows
a `limit` returned. Lists sort newest-first, so the rows past the cutoff are the oldest — the
longer a queue item goes unworked, the more likely it is to vanish. Local fixtures are smaller than
the limit, so it passes every test and reads as correct code (18 instances found 08/13/26).

Fetch rows scoped to what the section shows; get counts from a counts sibling — see
`count_orders_by_status` (`app/db/`) and the `/status-counts` routes, declared above any
`/{id}` route and mirroring the list function's source table. For a predicate that isn't a plain
status (`script_pdf_url IS NULL`, a date window), ask the list endpoint with `limit=1` and read
`total` rather than approximating. Review heuristic: if a number can't exceed the page size, it's
counting the buffer.

### Accessing client messages

[REDACTED: mail-access skill references and token scopes]


### Style notes 
- The questions the user asks are usually not rhetorical/leading.  The user actually wants to know the answer to the question out of curiosity.  When the user is asking a question, they are not necessarily insinuating that you've done something wrong. 
- If the user insists on an implementation that will break in production, results in poor UX, leaves out an important edge case, or you think is wrong in any way please speak up and push back.
