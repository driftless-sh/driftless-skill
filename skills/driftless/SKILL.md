---
name: driftless
description: Shared codebase memory for engineering teams. Use when the user mentions "driftless", "context memory", "scan diff", "architectural rules", "context watchers", "codebase drift", or asks to check code against team standards before pushing.
license: MIT
metadata:
  author: Driftless
  version: 1.1.0
  category: developer-tools
  tags: [architecture, context, nestjs, typescript, ai-agents, code-review, cli]
  documentation: https://driftless.icu/docs
---

# Driftless — Codebase Memory Layer

Driftless is the shared codebase memory layer for engineering teams using AI coding agents. It stores architectural context in the cloud and verifies local code against team rules.

## When to activate

- User mentions "driftless", "context", "scan", or "violations"
- Working in a repo with `AGENTS.md` that references Driftless
- Before modifying endpoints, guards, auth, services, or modules
- Before committing or pushing code changes

## Agent workflow

Follow this loop every time you work in a Driftless-enabled repo:

### Step 1: Load context before coding

```bash
driftless context get <feature>
```

Read the returned context: `what`, `how`, `where_files`, `decisions`, `gotchas`, `ownership`. Respect these constraints when writing code.

If unsure what feature to load:
```bash
driftless context list
driftless context search "keyword"
```

### Step 2: Implement changes

Write code following the patterns from the loaded context. Apply required decorators, guards, and conventions.

### Step 3: Scan diff before finishing

```bash
driftless scan --diff
```

This evaluates uncommitted changes against all active Cloud rules.

### Step 4: Fix violations

If violations are found, read each one and apply the fix described in the `explanation` field. Re-scan until clean.

Example violation:
```json
{
  "severity": "high",
  "message": "Every B2B endpoint must use BusinessAccessGuard",
  "explanation": "Missing required decorator: @UseGuards(BusinessAccessGuard)",
  "file": "src/routes/business/credit-line.ts",
  "line": 12
}
```

Fix:
```typescript
@Post('/business/credit-line')
@UseGuards(BusinessAccessGuard)  // added
async createCreditLine(@Body() dto: CreditLineDto) {
  return this.service.create(dto)
}
```

### Step 5: Push only when clean

Do not push if scan returns violations.

## Discovering and documenting context

When you learn something durable about the codebase that other agents should know:

```bash
driftless context add "feature-name" \
  --what "What this piece is" \
  --how "How it works" \
  --pattern "src/path/**" \
  --decisions "Why built this way" \
  --gotchas "Known traps" \
  --ownership "Who maintains it"
```

Fields:
- `--what` — what this piece is
- `--how` — how it works
- `--pattern` — glob for dynamic file resolution (preferred)
- `--where` — explicit file path (use instead of pattern for single files)
- `--decisions` — design decisions and rationale
- `--gotchas` — known edge cases and traps
- `--ownership` — team or person who maintains it

**Note:** `--pattern` and `--where` are mutually exclusive. Use `--pattern` for directories that may grow.

## Onboarding a new repo

```bash
driftless init
```

This scans the codebase (3 passes: identity, AST extraction, semantic enrichment), uploads the component map to Cloud, suggests architectural rules, and installs `AGENTS.md`.

## Rule types Driftless checks

| Pattern type | What it checks |
|---|---|
| `ENDPOINT_MISSING_DECORATOR` | Missing @UseGuards, @Roles, etc. |
| `FILE_IN_QUARANTINE` | New code in debt-heavy files |
| `FORBIDDEN_IMPORT` | Prohibited imports in new code |
| `MISSING_PROPERTY` | Required decorator properties |
| `DEPENDENCY_FORBIDDEN` | Forbidden cross-module calls |
| `COMPONENT_LIMIT` | File/method size limits |
| `NAMING_CONVENTION` | Component naming patterns |

## Common issues

**"Workspace not found"** — Run `driftless init` first to connect the repo.

**"No git remote found"** — Ensure the repo has a remote configured (`git remote -v`).

**"Watcher not found"** — Run `driftless context list` to see available watchers. The slug must match exactly.

**Scan returns nothing** — No uncommitted changes exist. Make sure you have unstaged modifications.

**API key errors** — Run `driftless login --key <key>` or set `DRIFTLESS_API_KEY` env var.

## Output format

All commands output JSON by default for agent consumption. Add `--human` for readable text:

```bash
driftless context get b2b-guard --human
driftless context search "auth" --human
```

## What Driftless does NOT do

- Does not modify code
- Does not approve/reject PRs or block merges
- Does not run tests or deploy
- Is not a CI pipeline

It is context memory + structural verification only.

## See also

- [CLI Command Reference](references/commands.md) — Full command docs with examples
- [Agent Workflow Guide](references/workflow.md) — Detailed workflow with patterns
- [Cloud Architecture](references/cloud.md) — How Cloud stores and syncs context
