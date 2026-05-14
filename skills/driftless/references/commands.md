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
```

Verifies: CLI auth, API connectivity, git workspace, repo connection, baseline, AGENTS.md, GitHub App.

Run this first when starting in a new environment or debugging a broken setup.

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
4. Generates architectural rules from detected patterns
5. Creates one context watcher per module (`created_by: driftless-init`)
6. Anchors existing docs: AGENTS.md, README.md, docs/*.md
7. Installs the Driftless AGENTS.md skill template

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

Suggested rules:
  • Every B2B endpoint must use BusinessAccessGuard
  • Services must not import controllers directly

Done.
```

---

### `driftless scan`

Scans local changes against Cloud rules. Exits `0` if clean, `1` if violations found.

```bash
# Scan staged + uncommitted changes
driftless scan

# Scan only uncommitted changes (pre-push standard)
driftless scan --diff

# Preview what would be reported without writing anything
driftless scan --dry-run
```

Output (clean):
```
Scanning uncommitted changes...
Clean — no violations detected.
```

Output (violations):
```
2 violation(s) found:

  [HIGH] Every B2B endpoint must use BusinessAccessGuard
    File: src/routes/business/credit-line.ts:12
    Code: @Post('/business/credit-line')

  [INFO] Quarantine: app.controller.ts
    File: src/app.controller.ts:205
    Code: newMethod() {
```

Use `exit $?` in CI to gate on the result.

---

### `driftless context`

Query and manage context watchers — the team's shared repo knowledge.

#### List all watchers

```bash
driftless context list
```

#### Get specific context

```bash
driftless context get <slug>
```

Returns: `what`, `how`, `where`, `used_by`, `gotchas`, `decisions`, `history` (recent events).

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

#### Create a watcher

```bash
driftless context add "<slug>" \
  --what "What this module does" \
  --how "How it is implemented" \
  --where "src/path/to/module/" \
  --gotchas "What to watch out for" \
  --decisions "Why it works this way"
```

#### Update an existing watcher

```bash
driftless context update <slug> \
  --gotchas "New gotcha discovered" \
  --decisions "Decision made in last PR"
```

Use this after discovering something worth persisting. Only pass the fields you want to update.

#### Anchor a file to a watcher

```bash
driftless context anchor <slug> --file path/to/file.md
```

Stores the file's content as `file_content` on the watcher. Useful for linking AGENTS.md sections, architecture docs, or runbooks.

#### Push context for specific files

```bash
driftless context push --files "src/auth/**"
```

Delivers context for all watchers that match the given file pattern. Use before starting work on a specific area.

#### Delete a watcher

```bash
driftless context delete <slug>
```

---

### `driftless session`

Brackets a working session for agents. Records which watchers were consulted and what changed.

```bash
# Start a session — loads relevant context, shows recent events
driftless session start

# Start with specific files in scope
driftless session start --files "src/billing/**"

# Finish session — scans diff, reports violations, summarizes context updates
driftless session finish
```

Use `session start` and `session finish` to wrap a complete agent task. The summary from `session finish` is useful as a handoff note for the next session.

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
| `0` | Clean — no violations, command succeeded |
| `1` | Violations found (scan) or command failed |
