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
4. Creates one context topic per module as a **suggestion** (`suggested: true`) — not real context yet
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

#### Sync a file to a watcher

```bash
driftless context sync <slug> --file path/to/file.md
```

Stores the file's content as `file_content` on the watcher. Useful for linking AGENTS.md sections, architecture docs, or runbooks.

#### Load context for specific files

```bash
driftless context load --files "src/auth/**"
```

Delivers context for all watchers that match the given file pattern. Use before starting work on a specific area.

#### Export watchers to YAML

```bash
driftless context export
driftless context export --dir .driftless/watchers
```

Writes one `.yaml` file per watcher to the specified directory (default: `.driftless/watchers/`). Creates the directory if it doesn't exist. Commit the output to your repo so context travels with the code.

```
Exported 12 watchers → .driftless/watchers/
```

#### Import watchers from YAML

```bash
driftless context import
driftless context import --dir .driftless/watchers
```

Reads all `*.yaml` files from the directory and creates or updates watchers in Cloud. Use to bootstrap a fresh workspace or restore context after a Cloud migration.

```
  ✓ billing (updated)
  ✓ auth (created)
  ✓ database (updated)
```

#### Delete a watcher

```bash
driftless context delete <slug>
```

---

### `driftless sync`

Pull Cloud state for the current repo. Run this at the start of every session — before touching any code.

```bash
driftless sync           # Cloud pull for current repo
driftless sync --json    # machine-readable output
```

Reports:
- Stale topics — code changed, context not updated
- Recent Cloud activity (FILE_CHANGED, UPDATED events)
- Suggested topics from init pending agent review

No local diff. Pure Cloud state. Use `driftless context get --diff` when you need topics for current local changes.

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

- `guards` empty AND `global_guards` empty → treat as **`no auth detected — verify`**, not "public" (may be a global APP_GUARD or composed auth decorator the scanner can't see).
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
