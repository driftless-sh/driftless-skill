---
name: driftless
description: Shared codebase memory layer for AI engineering teams. Load context, scan diffs, enforce rules, track watchers, keep agents aligned with team architecture.
version: 1.0.0
tags: [architecture, context, nestjs, typescript, ai-agents, code-review]
---

# Driftless — Agent Skill

## What Driftless is

Driftless is the **shared codebase memory layer** for engineering teams using AI coding agents.

It's not a linter. It's not a PR reviewer. It's not CI.

**Driftless Cloud** stores your team's architecture context — rules, watchers, component maps, violations. **The CLI** runs locally to scan, sync, and query that context. **Agents** use Driftless to load context before coding and verify changes before finishing.

## The agent's loop

Every time you work on a codebase with Driftless installed, follow this loop:

1. **Load context before touching code**
2. **Write your changes**
3. **Scan your diff before finishing**
4. **Fix violations if found**
5. **Push clean**

## Quick setup

```bash
# Install the CLI
npm install -g driftless

# Authenticate (get API key from your Driftless Dashboard)
driftless login --key drift_xxx

# Or use env var
export DRIFTLESS_API_KEY=drift_xxx
export DRIFTLESS_API_URL=https://your-instance.com/api/v1
```

## Before writing code

ALWAYS load relevant context first. Before touching endpoints, guards, auth, services, modules, routing, or package boundaries:

```bash
npx driftless context get <feature-name>
```

Example:
```bash
npx driftless context get b2b-guard
```

This returns:
- **what** the feature is
- **how** it works
- **where** the files live
- **who** consumes it
- **recent changes** (history)

If no watcher exists for what you need, check what's available:
```bash
npx driftless context list
```

## Before completing work

Always scan your diff against Cloud rules:

```bash
npx driftless scan --diff
```

If violations are reported, **fix them before pushing**. The scan output tells you:
- Which rule was violated
- What file and line
- What decorator/import/pattern is missing
- How to fix it

Example violation:
```
✕ B2B endpoints must use BusinessAccessGuard
  New endpoint is missing required decorator: @UseGuards(BusinessAccessGuard)
  File: src/routes/business/credit-line.ts:12
  Code: @Post('/business/credit-line')
```

## If you discover durable context

When you learn something about the codebase that future agents should know, document it:

```bash
npx driftless context add "b2b-guard" \
  --what "Central auth guard for all B2B endpoints" \
  --how "Validates API keys and Clerk identity for business-facing routes" \
  --where "src/shared/guards/business-access.guard.ts"
```

This becomes team memory in Cloud. Every agent can load it.

## Onboarding a new repo

If this repo doesn't have Driftless yet:

```bash
npx driftless init
```

This:
1. Connects the repo to your Driftless workspace
2. Scans the codebase → builds a component map
3. Suggests architectural rules based on detected patterns
4. Installs the Driftless AGENTS.md skill

## Rules — what gets checked

Driftless checks your code against **structural rules** stored in Cloud. Rules are defined by your team in plain language:

```
"Every B2B endpoint must use BusinessAccessGuard"
```

The engine translates this into deterministic checks. Structural rules never produce false positives — they use AST matching, not AI guesswork.

**7 pattern types**:
- `ENDPOINT_MISSING_DECORATOR` — missing @UseGuards, @Roles, etc.
- `FILE_IN_QUARANTINE` — code added to files marked as debt-heavy
- `FORBIDDEN_IMPORT` — prohibited imports in new code
- `MISSING_PROPERTY` — required decorator properties missing
- `DEPENDENCY_FORBIDDEN` — service depending on forbidden modules
- `COMPONENT_LIMIT` — file/method size limits
- `NAMING_CONVENTION` — components must follow naming patterns

## What Driftless does NOT do

- Does not modify your code
- Does not approve or reject PRs
- Does not block merges
- Is not a CI pipeline
- Does not run tests
- Does not deploy

It is **context memory + structural verification**. Nothing more.

## Important constraints

- **v1 is TypeScript backend only** (NestJS, Express, Next.js API routes)
- **Never hardcode model providers** — Driftless uses BYOM
- **All stored keys are encrypted** (AES-256-GCM)
- **Cloud is the source of truth** — not your local session memory
- **Context watchers auto-update** when tracked files change

## See also

- [CLI Command Reference](references/commands.md)
- [Agent Workflow Guide](references/workflow.md)
- [Cloud Architecture](references/cloud.md)
