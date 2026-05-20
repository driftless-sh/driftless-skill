# Agent Workflow Guide

## The Driftless Loop

Every time an AI agent works on a Driftless-enabled repo, it must follow this loop:

```
┌─────────────────────────────────────────────┐
│  1. SYNC CLOUD STATE                        │
│  driftless sync                             │
│  What drifted around your topics + team PRs │
├─────────────────────────────────────────────┤
│  2. LOAD CONTEXT                            │
│  driftless context get <feature>            │
│  Understand what/where/how before coding    │
├─────────────────────────────────────────────┤
│  3. CHECK THE CODE GRAPH                    │
│  driftless graph file <path>                │
│  Read entrypoints, callers, deps, guards    │
├─────────────────────────────────────────────┤
│  4. IMPLEMENT                               │
│  Write code following loaded context        │
├─────────────────────────────────────────────┤
│  5. UPDATE CONTEXT                          │
│  driftless context get --diff               │
│  Save durable discoveries before wrapping   │
└─────────────────────────────────────────────┘
```

## Step 1: Sync Cloud State

Run this first when starting or resuming:

```bash
driftless sync
```

Read **stale topics** (a topic whose covered code the team changed on a **tracked branch** since you last looked), **team PR activity**, the **tracking line** (which branches count as drift — the default branch is always tracked; `driftless branches` to view/change), and suggested topics. It is the deduped "what drifted around me" signal, not a raw event feed. If a topic is stale, inspect current code before relying on it. For your own *local* uncommitted changes use `driftless context get --diff` instead (no GitHub App needed).

## Step 2: Load Context

Before touching code that deals with endpoints, auth, services, modules, or package boundaries:

```bash
driftless context get <feature>
```

You get:
- `what` — description of the feature/concept
- `how` — implementation patterns in use
- `where` — canonical file paths
- `gotchas` — known traps
- `decisions` — why the team chose this shape
- `history` — recent changes and PRs that touched it

If no topic exists:

```bash
driftless context list
```

After you understand the area, create or update a topic.

## Step 3: Understand Blast Radius

Context is the team's intent. The code graph is deterministic reality:

```bash
driftless graph file <path-you-are-about-to-edit>
driftless graph impact --files "<path>"
```

Use it to answer:
- Which HTTP routes reach this code, and what guards protect them.
- Who calls it (`upstream`) vs. what it depends on (`downstream`).
- Which DTOs/contracts it accepts.

`guards` empty AND `global_guards` empty means `no auth detected — verify`, not "public".

## Step 4: Implement

Write code following the loaded context and verified graph facts. If the current implementation contradicts stored context, update the context after checking the code.

## Step 5: Update Context

Before wrapping up, load the context for your local diff:

```bash
driftless context get --diff
```

If you learned something durable, save it. **Batch every related change into ONE `update` call** — each PATCH bumps `version` and writes a history event, so N small updates pollute the trail:

```bash
driftless context update <slug> \
  --gotcha "Webhooks arrive out-of-order — idempotency key required" \
  --gotcha "Refund > 90 days breaks reconciliation" \
  --decision "Async webhook handler chosen to absorb retry storms" \
  --add-pattern "src/billing/webhooks/**"
```

Append flags (`--gotcha`, `--decision`, `--invariant`, `--check`) are repeatable and additive — every value lands atomically under a row lock, so concurrent agents don't lose appends.

If no topic covers the area, create one — multi-anchor with `--pattern` repeated keeps the topic narrow:

```bash
driftless context add "payment-module" \
  --what "Handles all payment processing via Stripe" \
  --how "PaymentsService → StripeAdapter; webhooks live in webhooks/" \
  --pattern "src/payments/**" \
  --pattern "src/billing/webhooks/**"
```

This becomes team memory. Every agent can query it with `driftless context get payment-module`.

## Anchoring discipline

Patterns claim a slice of the codebase. Healthy topics anchor **5–40 components**; anything past **100** is almost always over-broad — the topic devolves into a catch-all and `context get` returns generic guidance instead of the precise thing you needed. The CLI emits a non-blocking `⚠ over-broad anchor` warning when you cross the threshold; **split into narrower topics rather than suppress it**.

Good (narrow, conceptual):
```bash
driftless context add "checkout-flow" \
  --pattern "src/checkout/**" --pattern "src/cart/**"
```

Bad (catch-all that will rot):
```bash
driftless context add "backend" --pattern "src/**"   # don't
```

Manage patterns over time without clobbering the rest of the topic:

```bash
driftless context update billing-flow \
  --add-pattern "src/billing/refunds/**" \
  --remove-pattern "src/billing/legacy/**"
```

Both are idempotent (re-running is a no-op).

## Linking topics with `[[slug]]`

Topics are not islands. When something only makes sense in the context of another topic, link instead of duplicate. Inside any free-text field (`--what`, `--how`, `--decisions`, `--gotcha`, `--invariant`, `--ownership`), write `[[other-topic-slug]]` and the API records the link automatically on every create/update.

```bash
driftless context update refund-flow \
  --gotcha "Same race as [[stripe-webhook-ingest]]" \
  --decision "Idempotency-key derived from charge_id, mirroring [[payment-gateway]]"
```

`context get refund-flow` then shows a `references` section listing the forward links; `context get payment-gateway` shows `referenced by: refund-flow`. The dashboard renders each `[[slug]]` as a clickable link.

**When to link:**
- A gotcha only makes sense if you already know another topic — link instead of restating.
- A decision is the consequence of an earlier decision recorded elsewhere — link.
- Two topics describe parts of the same flow — link both directions.

**When NOT to link:**
- The slug just sounds vaguely related — don't.
- The target topic doesn't exist yet — either create it first, or skip the link. Dead links sit silently in the data and never produce a backlink, so the trace rots invisibly.

Slug grammar: lowercase alphanumerics + hyphens, must start with `[a-z0-9]`. Self-references are stripped automatically.
