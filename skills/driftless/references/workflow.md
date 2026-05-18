# Agent Workflow Guide

## The Driftless Loop

Every time an AI agent works on a Driftless-enabled repo, it must follow this loop:

```
┌─────────────────────────────────────────────┐
│  1. SYNC CLOUD STATE                        │
│  driftless sync                             │
│  Catch stale topics and recent repo events  │
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

Read stale topics, recent `FILE_CHANGED` / `UPDATED` events, PR observations, and suggested topics. If a topic is stale, inspect current code before relying on it.

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

If you learned something durable, save it:

```bash
driftless context update <slug> \
  --gotcha "What I learned that was not documented" \
  --decisions "Why the team does it this way"
```

If no topic covers the area:

```bash
driftless context add "payment-module" \
  --what "Handles all payment processing via Stripe" \
  --how "src/payments/payments.module.ts -> PaymentsService -> StripeAdapter" \
  --where "src/payments/"
```

This becomes team memory. Every agent can query it with `driftless context get payment-module`.
