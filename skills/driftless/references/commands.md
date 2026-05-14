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
4. Generates architectural rules from detected patterns (marked as `auto_suggested`)
5. Creates one context topic per module as a **suggestion** (`suggested: true`) — not real context yet
6. Detects existing docs and reports them — does NOT auto-sync them (syncing is an intentional agent action)
7. Installs the Driftless skill into CLAUDE.md and AGENTS.md

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

### `driftless sync`

Pull Cloud state for the current repo. Run this at the start of every session — before touching any code.

```bash
driftless sync           # Cloud pull for current repo
driftless sync --json    # machine-readable output
```

Reports:
- Stale topics — code changed, context not updated
- Recent Cloud activity (FILE_CHANGED, UPDATED events)
- Open violations for this repo
- Suggested topics from init pending agent review

No local diff. No scan. Pure Cloud state. Use `driftless scan --diff` separately before pushing.

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

Query and manage context topics — the team's shared repo knowledge.

#### List all topics

```bash
driftless context list

# Show init-generated suggestions pending agent review
driftless context list --suggested
```

#### Get specific context

```bash
driftless context get <slug>
```

Returns: `what`, `how`, `where`, `gotchas`, `decisions`, `history` (recent events).

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

#### Create a context topic

```bash
driftless context add "<slug>" \
  --what "What this module does" \
  --how "How it is implemented" \
  --where "src/path/to/module/" \
  --gotchas "What to watch out for" \
  --decisions "Why it works this way"
```

#### Update an existing topic

```bash
driftless context update <slug> \
  --gotchas "New gotcha discovered" \
  --decisions "Decision made in last PR"
```

Every `context update` call automatically links the current repo to the topic. If the same concept spans multiple repos, run `context update` from each one — the topic accumulates all repos in `where_repos`. No flag needed.

#### Sync a doc to a topic

```bash
driftless context sync <slug> --doc path/to/file.md
```

Uploads the full content of the doc to Cloud and links it to the topic. When files the topic tracks change, Driftless knows exactly which doc is out of sync.

#### Load context for specific files

```bash
driftless context load --files "src/auth/**"
```

Delivers full content for all context topics that match the given file pattern. Use before starting work on a specific area.

#### Export topics to YAML

```bash
driftless context export
driftless context export --dir .driftless/watchers
```

Writes one `.yaml` file per topic to the specified directory. Commit the output so context travels with the code.

#### Import topics from YAML

```bash
driftless context import
driftless context import --dir .driftless/watchers
```

Reads all `*.yaml` files from the directory and creates or updates topics in Cloud.

#### Delete a topic

```bash
driftless context delete <slug>
```

---

### `driftless install-skill`

Writes the Driftless skill into CLAUDE.md and AGENTS.md so AI agents (Claude Code, Cursor, Codex) pick it up automatically. Runs automatically at the end of `driftless init`.

```bash
driftless install-skill
```

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
