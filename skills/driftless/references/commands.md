# Driftless CLI — Command Reference

## Installation

```bash
npm install -g @driftless-sh/cli
```

## Authentication

Driftless CLI uses API keys. No browser OAuth. No session management.

**Priority order:**
1. `DRIFTLESS_API_KEY` environment variable
2. `~/.driftless/config.json` file

```bash
# Save key to config file
driftless login --key drift_xxx

# Or via environment variable (recommended for CI and agents)
export DRIFTLESS_API_KEY=drift_xxx
```

Get your API key at driftless.icu → Settings → API Keys.

---

## Commands

### `driftless doctor`

Checks that the environment is fully configured before doing anything else.

```bash
driftless doctor
driftless doctor --json   # machine-readable: { ok, summary, checks[], workspace_diagnostics? }
```

Verifies: CLI auth, API connectivity, git workspace, repo connection, baseline, AGENTS.md, GitHub App. The Workspace line shows the resolved slug and its source (`config` cache, `/me`, or `git-org`). On failure it prints structured diagnostics (tried slugs, `/me` status, failed endpoint).

Run this first when starting in a new environment or debugging a broken setup. `--json` for agents/CI that parse the result.

---

### `driftless init`

Connects a repo to Driftless Cloud and bootstraps context from what already exists. Run once per repo from the repo root.

```bash
driftless init
```

What it does:
1. Detects git remote → finds or creates workspace + repo record
2. Scans the codebase → builds component map (controllers, services, guards, modules, DTOs)
3. Uploads baseline to Cloud
4. **Only with `--suggest`** (opt-in, off by default): creates one suggested topic per module (`suggested: true`) — drafts, not real context yet. Without the flag, no topics are created.
5. Detects existing docs and reports them — does NOT auto-sync them (syncing is an intentional agent action)
6. Installs the Driftless skill into CLAUDE.md and AGENTS.md

Output example:
```
Repository: nippy-tech/profile_nippy_ms
Workspace: nippy-tech (existing)
Repo: profile_nippy_ms (connected)

Scanning codebase...
  Framework:    nestjs
  Type:         api
  Auth:         cognito, clerk, jwt, apikey
  Endpoints:    155
  Guards:       15
  Services:     58
  Modules:      17

Done.
```

---

### `driftless context`

Query and manage context topics — the team's shared repo knowledge.

#### List all topics

```bash
driftless context list
driftless context list --suggested   # show auto-generated draft topics
driftless context list --stale      # topics whose anchored code changed
driftless context list --kind code-context
```

#### Get specific context

```bash
driftless context get <slug>
```

Returns: `what`, `how`, `where`, `used_by`, `gotchas`, `decisions`, `content`, `ownership`, `invariants`, `required_checks`, `history` (recent events).

Example:
```bash
driftless context get billing
```

```
Name:      billing
What:      Handles all subscription and payment processing
How:       BillingModule → BillingService → StripeAdapter. Never call Stripe directly.
Where:     src/billing/
Gotchas:   Stripe webhook events must be idempotent — retries happen. Always check event.id before processing.
Decisions: We use Stripe's customer portal instead of building our own to reduce PCI scope.
Updated:   2026-05-10 (FILE_CHANGED: src/billing/billing.service.ts)
```

#### Search by topic

```bash
driftless context search <query>
```

Use when you don't know the exact slug.

```bash
driftless context search "payment"
→ billing, payment-gateway, stripe-adapter
```

#### Create a topic

```bash
driftless context add "<slug>" \
  --content "# Module Name\n\n## What\nWhat this module does\n\n## How\nHow it is implemented" \
  --pattern "src/auth/**" \
  --pattern "src/users/**" \
  --kind code-context \
  --tags auth,security \
  --rel depends_on:token-refresh
```

`--pattern` is **repeatable** — pass it multiple times for a multi-anchor topic. The CLI blocks creation if any pattern matches 0 files locally; emits a non-blocking `⚠ over-broad anchor` warning when the topic covers more than 100 components (healthy is 5–40 — see "Anchoring discipline" below).

Other `context add` flags: `--what`, `--how`, `--where` (single file path), `--decisions`, `--gotchas`, `--ownership`, `--status <reviewed|draft>`, `--file` (content from file path), `--dry-run`.

For backward compat `--where path/to/file` still works as a single explicit anchor. Use `--pattern` for globs (the common case).

#### Update an existing topic

```bash
# Replace scalars
driftless context update <slug> --what "..." --how "..."

# Append, never clobber — repeatable
driftless context update <slug> \
  --gotcha "Discovered a new trap" \
  --gotcha "And another one" \
  --decision "Chose A over B because Y" \
  --invariant "All writes must use the lock" \
  --check "Run integration suite before merging"

# Manage patterns atomically — repeatable
driftless context update <slug> \
  --add-pattern "src/billing/webhooks/**" \
  --add-pattern "src/billing/refunds/**" \
  --remove-pattern "src/billing/legacy/**"
```

- **Singular** flag (`--gotchas`, `--decisions`, `--pattern`) = REPLACE the whole field.
- **Append** flags (`--gotcha`, `--decision`, `--invariant`, `--check`) = APPEND a new line. Pass each as many times as you want — every value is appended atomically in a single PATCH (the SQL UPDATE runs under a row lock so concurrent appends don't lose data).
- `--add-pattern` / `--remove-pattern` are idempotent: running the same one twice is a no-op. Both can fire in the same PATCH — remove applies first, then add.
- Every PATCH bumps `version` and writes a history event, so **batch related changes into ONE invocation** rather than splitting into N small ones.

Other `context update` flags: `--content` (full markdown body), `--kind`, `--tags`, `--status <reviewed|draft>`, `--where` (single file path), `--gotchas` (replace all gotchas), `--decisions` (replace all decisions), `--ownership`, `--pattern` (replace all patterns), `--enforce "rule"`, `--dry-run`, `--json`, `--check` (required check, repeatable).

Get `--help` on any subcommand for the full flag list:

```bash
driftless context add --help
driftless context update --help
```

#### Anchoring discipline

Patterns define which slice of the codebase a topic claims. Size matters:

| Component count | Verdict |
|---|---|
| 5–40 | Healthy — narrow enough to mean something specific |
| 41–99 | Wide — verify it's truly one concept |
| 100+ | Trap — almost always over-broad; the topic becomes a useless catch-all |

The CLI emits `⚠ over-broad anchor` after `add`/`update` when the topic crosses 100 components. Don't suppress it — split into narrower topics instead.

**Good** (narrow, multi-anchor):
```bash
driftless context add "checkout-flow" \
  --pattern "src/checkout/**" --pattern "src/cart/**" \
  --what "Cart → checkout → payment"
```

**Bad** (catch-all that will rot):
```bash
driftless context add "backend" --pattern "src/**"       # don't
```

#### Linking topics with `[[slug]]`

Inside any free-text field on a topic (`--what`, `--how`, `--decisions`, `--gotcha`, `--invariant`, `--ownership`), write `[[other-topic-slug]]` to mark a forward reference. The API parses every mention on every create/update and stores the slugs in `references_topics`. The reverse direction (`referenced_by`) is computed at read-time.

```bash
driftless context update refund-flow \
  --gotcha "Same race as [[stripe-webhook-ingest]]" \
  --decision "Idempotency-key derived from charge_id, mirroring [[payment-gateway]]"
```

`context get refund-flow` then shows the forward refs; `context get payment-gateway` shows `refund-flow` under `referenced by`.

- Slug grammar: lowercase alphanumerics + hyphens, must start with `[a-z0-9]`. Anything else is ignored.
- Self-references are silently stripped.
- Dead links (slugs that don't resolve to a real topic) are accepted in writes but never produce a backlink on the other side — they sit silently in `references_topics` until you fix them.

#### Cross-repo: link a topic to another repo

```bash
driftless context link <slug>
```

Registers the repo you are currently in into the topic's `where_repos`. Touches only the repo list — `what`/`how`/`gotchas`/`decisions` are NOT modified. Use this when the same concept lives in another repo.

`context get <slug>` then shows `used in (N repos): …` and groups components by repo. `context update` also auto-links the repo you run it from as a side effect — use `context link` when linking is all you want.

#### Sync a file or note to a topic

```bash
# Anchor a doc file — sets file_content, origin=doc, status=draft, kind=docs-note
driftless context sync <slug> --doc path/to/file.md

# Add a quick note — maps to the decisions field
driftless context sync <slug> --note "Key finding from recent review"

# Sync with path anchoring
driftless context sync <slug> --doc path/to/file.md --files "src/auth/**"
```

Stores the file's content as `file_content` on the topic. Useful for linking AGENTS.md sections, architecture docs, or runbooks.

#### Load context for specific files

```bash
# Takes comma-separated file paths (not globs)
driftless context load --files "src/auth/guard.ts,src/auth/service.ts"
```

Delivers context for all topics that match the given file paths. Use before starting work on a specific area. For local uncommitted changes, use `context get --diff` instead.

#### Delete a topic

```bash
driftless context delete <slug>
driftless context delete <slug> --dry-run   # preview without deleting
```

---

### `driftless sync`

Pull Cloud state for the current repo. Run this at the start of every session — before touching any code.

```bash
driftless sync           # Cloud pull for current repo
driftless sync --json    # machine-readable output
```

Reports — the deduped "what moved around my topics" signal, not a raw event feed:
- **Stale topics** — a topic whose covered code the team changed on a tracked branch since you last looked
- **Team PR activity** — what teammates shipped against this repo's context (via the GitHub App)
- **Tracking line** — which branches' pushes count as drift (the default branch is always tracked)
- Suggested topics from `init --suggest` pending review

No local diff. Pure Cloud state. Use `driftless context get --diff` when you need topics for your current *local* uncommitted changes. `context load` requires `--files` and does not support `--diff`.

---

### `driftless branches`

View or set which branches' pushes count as drift. The repo's **default branch** (from GitHub, auto-detected — main/master/whatever) is always tracked: zero config. Add extra branches only if your real integration branch isn't the GitHub default (e.g. you ship from `release` or `develop`).

```bash
driftless branches                 # show tracked branches
driftless branches add release     # also detect drift from pushes to `release`
driftless branches rm staging
driftless branches --json
```

A push to a non-tracked branch is recorded for audit but does **not** mark topics stale — a throwaway feature branch never creates false drift. This is why `sync` only surfaces drift from tracked branches.

---

### `driftless graph`

Deterministic code graph from the scanned components — no LLM. Run before editing a file to know its blast radius.

```bash
# Entrypoints (routes that reach it, with guards), upstream callers,
# downstream deps, contracts (DTOs). Accepts repo-root OR scan-root paths.
driftless graph file src/billing/billing.service.ts
driftless graph file src/billing/billing.service.ts --json --depth 4

# Blast radius: upstream consumers + reachable endpoints a change can break.
driftless graph impact --files "src/billing/billing.service.ts"
driftless graph impact --files "a.ts,b.ts" --direction both --json
```

Key fields: `entrypoints[]` (`method`, `path`, `handler`, `guards`, `guards_detected`), `upstream`, `downstream`, `contracts`, `global_guards` (repo-wide APP_GUARD), and per-component `confidence` + `evidence` (`file:line`).

- `guards` populated:
  - NestJS — the `@UseGuards(X)` name (e.g. `"AuthGuard"`); `APP_GUARD` providers go in `global_guards`.
  - Next.js — `"clerk-auth"` when `auth()` / `currentUser()` from `@clerk/nextjs/server` is invoked in the file. Call-site `file:line` lives in `metadata.guard_evidence`.
- `guards` empty AND `global_guards` empty → treat as **`no auth detected — verify`**, not "public" — may still be a composed/custom auth decorator or non-Clerk auth helper the scanner can't see.
- `downstream` (Next.js/React) follows resolved `imports` edges: a page lists the `react-component`s it renders, a component lists the `react-hook`s it uses. Components/hooks are detected by shape (JSX / `use*` naming), and imports resolve file-locally (tsconfig paths + relative). Imports of untracked files (libs, services, node_modules) resolve to nothing by design.
- `graph impact` default `--direction consumers` = what breaks; `dependencies` = what it uses; `both` = both.
- `--depth N` 1–6 (default 3). `--json` for agents/CI.

---

### `driftless context doctor`

Audits the context layer itself — answers "can I trust `context get`?".

```bash
driftless context doctor
driftless context doctor --json
```

Categories: `stale` (code changed, context not updated), `orphaned` (repo deleted — hidden from agents), `draft` (suggested, never confirmed), `docs_pending` (doc anchored, never synced), `repo_leak` (references an unknown repo id). Exits non-zero when `orphaned` or `repo_leak` are present.

---

### `driftless install-skill`

Writes the Driftless AGENTS.md template into the current repo so AI agents (Claude Code, Cursor, Codex) pick it up automatically.

```bash
driftless install-skill
```

Creates or appends to `AGENTS.md` at the repo root.

---

## Environment variables

| Variable | Default | Description |
|---|---|---|
| `DRIFTLESS_API_KEY` | — | API key for authentication |
| `DRIFTLESS_API_URL` | `http://localhost:3000/api/v1` | Cloud API endpoint |

---

## Exit codes

| Code | Meaning |
|---|---|
| `0` | Command succeeded |
| `1` | Command failed |
