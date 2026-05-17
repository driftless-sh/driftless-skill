# Agent Workflow Guide

## The Driftless loop

Every time an AI agent works on a Driftless-enabled repo, it must follow this loop:

```
┌─────────────────────────────────────────────┐
│  1. LOAD CONTEXT                            │
│  driftless context get <feature>            │
│  Understand what/where/how before coding    │
├─────────────────────────────────────────────┤
│  2. IMPLEMENT                               │
│  Write code following the loaded context    │
│  Apply required decorators and patterns     │
├─────────────────────────────────────────────┤
│  3. SCAN DIFF                               │
│  driftless scan --diff                      │
│  Check changes against Cloud rules          │
├─────────────────────────────────────────────┤
│  4. FIX VIOLATIONS                          │
│  Read violation explanation                 │
│  Apply required fixes                       │
│  Re-scan until clean                        │
├─────────────────────────────────────────────┤
│  5. PUSH                                    │
│  Only after clean scan                      │
│  Cloud updates watchers on push             │
└─────────────────────────────────────────────┘
```

## Step 1: Load Context

**ALWAYS do this first.** Before touching any code that deals with:

- Endpoints (controllers, routes, handlers)
- Auth (guards, middleware, decorators)
- Services (business logic, dependencies)
- Modules (imports/exports structure)
- Package boundaries (what imports what)

**How:**
```bash
driftless context get <feature>
```

**What you get:**
- `what` — description of the feature/concept
- `how` — how it's implemented, patterns used
- `where` — canonical file paths where it lives
- `used_by` — what consumes this feature
- `history` — recent changes and PRs that touched it

**If no watcher exists:**
```bash
driftless context list    # see what's available
```

If what you need isn't documented yet, use `driftless context add` to create it after you understand it.

## Step 1.5: Understand the blast radius

Context is the team's intent. The code graph is the deterministic reality — run it before editing the file:

```bash
driftless graph file <path-you-are-about-to-edit>
driftless graph impact --files "<path>"   # what breaks if you get it wrong
```

Use it to answer, before writing a line:
- Which HTTP routes reach this code, and **what guards protect them** (`guards` empty AND `global_guards` empty ⇒ `no auth detected — verify`, never assume public).
- Who calls it (`upstream`) vs. what it depends on (`downstream`).
- Which DTOs/contracts it accepts.

`confidence` < 1 or surprising results → open the `evidence` `file:line` and verify. Don't trust the graph blindly; it's a fast map, not a substitute for reading the code you're about to change.

## Step 2: Implement

Write your code. Apply the patterns you learned from context:

```typescript
// ❌ WRONG — no guard on B2B endpoint
@Post('/business/credit-line')
async createCreditLine(@Body() dto: CreditLineDto) {
  return this.service.create(dto)
}

// ✅ CORRECT — BusinessAccessGuard applied
@Post('/business/credit-line')
@UseGuards(BusinessAccessGuard)
async createCreditLine(@Body() dto: CreditLineDto) {
  return this.service.create(dto)
}
```

## Step 3: Scan Diff

Before declaring work complete:

```bash
driftless scan --diff
```

**What gets checked:**
- All active structural rules in your workspace
- New code against the component baseline (known debt is ignored)
- Both staged and unstaged changes (or `--diff` for unstaged only)

**Severity levels:**
- `critical` / `high` — **must fix** before pushing
- `warning` — should fix, strong recommendation
- `info` — advisory, semantic suggestions

## Step 4: Fix Violations

Read each violation carefully:

```
[HIGH] Every B2B endpoint must use BusinessAccessGuard
  New endpoint is missing required decorator: @UseGuards(BusinessAccessGuard)
  File: src/routes/business/credit-line.ts:12
  Code: @Post('/business/credit-line')
```

**How to fix:**
1. Look at the `explanation` field — it tells you exactly what's missing
2. The `Code` snippet shows the violating line
3. Add the required decorator/pattern
4. Re-scan: `driftless scan --diff`

```typescript
// Add the missing guard
@Post('/business/credit-line')
@UseGuards(BusinessAccessGuard)  // ← Added
async createCreditLine(@Body() dto: CreditLineDto) {
  return this.service.create(dto)
}
```

## Step 5: Push

**Only push when scan returns clean.**

After push, Cloud automatically updates context watchers if you modified tracked files. No manual updates needed.

## When you discover new context

If you learn something durable about the codebase that other agents should know:

```bash
driftless context add "payment-module" \
  --what "Handles all payment processing via Stripe" \
  --how "src/payments/payments.module.ts → PaymentsService → StripeAdapter" \
  --where "src/payments/"
```

This becomes team memory. Every agent can query it with `driftless context get payment-module`.

## Common patterns

### NestJS endpoints
- B2B endpoints require `@UseGuards(BusinessAccessGuard)`
- Admin endpoints require `@UseGuards(AdminGuard)`
- Public endpoints may have no guard (depending on rules)

### Quarantined files
Files marked as quarantined (`FILE_IN_QUARANTINE` rule) have known technical debt. **Do not add new code to them.** Refactor the debt or extract new logic to separate files.

### Forbidden imports
Some modules are forbidden from being imported in new code. The scan will tell you which import is forbidden and suggest alternatives.

### Naming conventions
Components must follow team naming patterns. For example, all services must end with `Service`, all guards must end with `Guard`.
