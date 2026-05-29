# Agent Workflow Guide

## The Driftless Loop

Every time an AI agent works on a Driftless-enabled repo, it follows this loop:

```text
1. SYNC CLOUD STATE
   driftless sync
   Read drift, stale topics, PR activity, and tracked branches.

2. RETRIEVE TOPIC CONTEXT
   driftless context get <slug>
   driftless context get --files "path/to/file.ts"
   Understand what/how/gotchas/decisions before editing.

3. IMPLEMENT
   Make the change using the retrieved topic context.

4. UPDATE TOPICS
   driftless context update <slug> --gotcha "..." --decision "..."
   Persist durable discoveries.

5. REFRESH BEFORE FINISHING
   driftless sync
   driftless context get --diff
   Confirm the local diff lines up with current context.
```

## Step 1: Sync Cloud State

Run this first when starting or resuming:

```bash
driftless sync
```

Read stale topics, team PR activity, and the tracking line. Drift is scoped to tracked branches: the default branch is always tracked, and teams can add extra branches with `driftless branches`.

If a topic is stale, inspect current code before relying on old assumptions. For your own local uncommitted changes, use `driftless context get --diff`.

## Step 2: Retrieve Topic Context

If you know the topic:

```bash
driftless context get <slug>
```

If you know the files:

```bash
driftless context get --files "src/auth/guard.ts,src/auth/service.ts"
```

Read:

- `what` — what this area does
- `how` — implementation patterns
- `gotchas` — traps and edge cases
- `decisions` — why the team chose this shape
- `invariants` — what must stay true
- `required_checks` — what to run or inspect before finishing
- `anchors` — the files, patterns, and repos the topic claims

If no topic exists, inspect enough code to understand the area, then create a topic or add an anchor to the closest existing one.

## Step 3: Implement

Write code following the retrieved context. If the current implementation contradicts stored context, treat that as a context drift signal: verify reality, then update the topic.

## Step 4: Update Topics

If you learned something durable, save it. Batch related changes into one update:

```bash
driftless context update <slug> \
  --gotcha "Webhooks arrive out-of-order — idempotency key required" \
  --decision "Async handler chosen to absorb retry storms" \
  --invariant "Webhook handlers must be idempotent" \
  --check "Replay duplicate event ids before merging" \
  --add-pattern "src/billing/webhooks/**"
```

Append flags (`--gotcha`, `--decision`, `--invariant`, `--check`) are additive. Prefer them over plural replace flags unless you intentionally want to replace the whole field.

If no topic covers the area:

```bash
driftless context add payment-module \
  --kind reference \
  --what "Handles payment processing" \
  --how "PaymentsService delegates to StripeAdapter; webhooks live in webhooks/" \
  --pattern "src/payments/**" \
  --pattern "src/billing/webhooks/**"
```

## Step 5: Refresh Before Finishing

Before wrapping up:

```bash
driftless sync
driftless context get --diff
```

If `--diff` matches a stale topic that your change touches, update it before pushing.

## Anchoring Discipline

Patterns claim a slice of the codebase. Healthy topics anchor about 5-40 files. Anything above 100 is usually too broad.

Good:

```bash
driftless context add checkout-flow \
  --pattern "src/checkout/**" \
  --pattern "src/cart/**"
```

Bad:

```bash
driftless context add backend --pattern "src/**"
```

Use `--add-pattern` and `--remove-pattern` to evolve anchors without clobbering the rest of the topic:

```bash
driftless context update billing-flow \
  --add-pattern "src/billing/refunds/**" \
  --remove-pattern "src/billing/legacy/**"
```

## Linking Topics

Use `[[slug]]` in free-text fields when a gotcha or decision depends on another topic:

```bash
driftless context update refund-flow \
  --gotcha "Same idempotency race as [[stripe-webhook-ingest]]"
```

Use typed relations for strong graph edges:

```bash
driftless context update refund-flow --rel depends_on:stripe-webhook-ingest
driftless context relations refund-flow
driftless context graph refund-flow
```

## PR Bot Loop

Every PR comment shows:

- Topics matched by changed files.
- Decisions and gotchas relevant to review.
- Files outside any topic.
- Suggested commands to add anchors or create topics.

Use those gaps to grow the topic layer over time.
