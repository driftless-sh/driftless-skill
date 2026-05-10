# Agent Workflow Guide

## The loop

```
1. LOAD CONTEXT  →  driftless context get <feature>
2. IMPLEMENT     →  Write code following loaded context
3. SCAN DIFF     →  driftless scan --diff
4. FIX           →  Resolve violations, re-scan
5. PUSH          →  Only when clean
```

## Step 1: Load context

Do this before touching: endpoints, guards, auth, services, modules, or package boundaries.

```bash
driftless context get <feature>
```

Returns: `what`, `how`, `where_files`, `pattern`, `decisions`, `gotchas`, `ownership`, `where_used_by`, `history`.

If no watcher exists for your feature:
```bash
driftless context list
driftless context search "keyword"
```

## Step 2: Implement

Follow the patterns from context. Respect `decisions` and avoid `gotchas`.

```typescript
// Wrong — missing guard
@Post('/business/credit-line')
async createCreditLine(@Body() dto: CreditLineDto) {
  return this.service.create(dto)
}

// Correct — guard applied
@Post('/business/credit-line')
@UseGuards(BusinessAccessGuard)
async createCreditLine(@Body() dto: CreditLineDto) {
  return this.service.create(dto)
}
```

## Step 3: Scan

```bash
driftless scan --diff
```

Checks: active structural rules, new code vs baseline (known debt ignored).

Severity: `critical`/`high` = must fix, `warning` = should fix, `info` = advisory.

## Step 4: Fix violations

Read the `explanation` field — it tells you exactly what's missing. The `file` and `line` fields point to the violation. Re-scan after each fix.

## Step 5: Push

Only push when `{ "status": "clean", "violations": [] }`.

## Documenting new context

When you discover durable info about the codebase:

```bash
driftless context add "payment-module" \
  --what "Handles Stripe payments" \
  --how "PaymentsService → StripeAdapter" \
  --pattern "src/payments/**" \
  --decisions "Stripe due to existing integration" \
  --gotchas "Idempotency keys required" \
  --ownership "@payments-team"
```

## Pattern vs where_files

- `--pattern` = glob rule, resolves dynamically (preferred for directories)
- `--where` = explicit file path (for single files)
- Mutually exclusive — use one or the other
- Pattern resolves to `where_files` on scan

## Output format

JSON by default (for agents). `--human` for readable text (for debugging).

```bash
driftless context get b2b-guard        # JSON
driftless context get b2b-guard --human  # readable
```
