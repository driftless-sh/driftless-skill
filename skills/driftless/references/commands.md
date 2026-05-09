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

### Login

```bash
# Interactive (saves to ~/.driftless/config.json)
driftless login --key drift_xxx

# Via environment variable (recommended for CI/agents)
export DRIFTLESS_API_KEY=drift_xxx
export DRIFTLESS_API_URL=https://your-instance.com/api/v1
```

Get your API key from the Driftless Dashboard → API Keys section.

---

## Commands

### `driftless init`

Connects a repo to Driftless Cloud. Runs from the repo root.

```bash
driftless init
```

What it does:
1. Detects git remote → finds or creates workspace + repo
2. Clones and scans the codebase → builds component map
3. Stores baseline in Cloud
4. Suggests architectural rules based on detected patterns
5. Installs Driftless AGENTS.md skill

Output:
```
Repository: nippy-tech/profile_nippy_ms
Workspace: nippy-tech (existing)
Repo: profile_nippy_ms (connected)

Scanning codebase...
Results:
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

Done! Run 'driftless scan --diff' before pushing to check for violations.
```

---

### `driftless scan`

Scans local changes against Cloud rules.

```bash
# Scan all staged + uncommitted changes
driftless scan

# Scan only uncommitted changes (not staged)
driftless scan --diff
```

What it does:
1. Gets git diff from current working directory
2. Auto-detects workspace from git remote
3. Sends diff to Cloud → evaluates against active rules
4. Returns violations or clean status

Output (clean):
```
Scanning uncommitted changes...
Clean — no violations detected.
```

Output (violations):
```
Scanning uncommitted changes...

2 violation(s) found:

  [HIGH] Every B2B endpoint must use BusinessAccessGuard
    New endpoint is missing required decorator: @UseGuards(BusinessAccessGuard)
    File: src/routes/business/credit-line.ts:12
    Code: @Post('/business/credit-line')

  [INFO] Quarantine: app.controller.ts
    New code added to quarantined file: src/app.controller.ts. This file has known debt and should not grow.
    File: src/app.controller.ts:205
    Code: newMethod() {

Fix these violations before pushing.
```

---

### `driftless context`

Query and manage context watchers — the team's shared architectural memory.

```bash
# List all watchers
driftless context list

# Get specific context
driftless context get <name>
# Example: driftless context get b2b-guard

# Create new watcher
driftless context add "b2b-guard" \
  --what "Central auth guard for all B2B endpoints" \
  --how "Validates API keys and Clerk identity" \
  --where "src/shared/guards/business-access.guard.ts"
```

Output from `context get b2b-guard`:
```
Name:     B2B Guard
What:     Central auth guard for all B2B endpoints
How:      Validates API keys and Clerk identity for business-facing routes
Where:    src/shared/guards/business-access.guard.ts
Used by:  AppController, ConsoleController, BusinessController
Updated:  2026-05-09 (3 PRs ago)
```

---

### `driftless install-skill`

Installs the Driftless AGENTS.md template into the current repo.

```bash
driftless install-skill
```

This creates or appends to `AGENTS.md` with the Driftless instructions that AI agents (Claude Code, Codex, Cursor) can read.

---

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `DRIFTLESS_API_KEY` | — | API key for authentication |
| `DRIFTLESS_API_URL` | `http://localhost:3000/api/v1` | Cloud API endpoint |
