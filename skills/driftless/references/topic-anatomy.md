# Topic Anatomy

A Driftless topic is a markdown note with structured fields and optional code anchors. This reference covers every field and the semantics of how updates merge.

## Shape — content-first

Write the topic as markdown `--content`: a clear, readable explanation, like a
good doc. The content body IS the topic. The structured fields are **optional
highlights** layered on top:

- `--what` — one-sentence summary (shown first in lists and `context get`).
- `--how` — a SHORT (1–3 sentence) note on the approach/mechanism. NOT a place
  for a document — the long-form goes in `--content` (where it renders as
  markdown). A doc pasted into `--how` is flagged by `context doctor` as
  `mis-shaped how`.
- `--gotcha` / `--decision` / `--invariant` / `--check` — add one only when you
  want that specific thing surfaced to the machine (the PR bot, a future agent's
  brief) *on top of* the content. Never invent an empty gotcha or decision just
  because the flag exists.
- `--tags` — free-form labels to group and filter topics.
- `--pattern` — anchors are a separate axis (file matching); always worth setting.

Three starter templates ship at `.driftless/assets/templates/` — `reference.md`,
`decision.md`, and `runbook.md` — as optional content scaffolding. Use one if it
helps; the content is what matters, not the form.

Agent prompt guidance:

```text
Write every topic as markdown --content. The content body is the topic.

Use --what as a one-sentence summary.
Use --tags to label and group topics.
Add --gotcha / --decision / --invariant / --check only when you want that
specific thing surfaced on top of the content — never to fill a form.
```

## Fields

### Required-ish (most topics have these)

- **`name`** (slug) — lowercase kebab-case. Set on `add`, immutable thereafter.
- **`what`** — one-sentence summary. Surfaces first in `context get`.
- **`how`** — implementation patterns. Avoid restating code; capture *intent* and *structure*. Keep it SHORT (1–3 sentences); long-form goes in `content`, not here.

### Optional but high-value

- **`where`** — single canonical file path. For multi-file scopes use anchors (`--pattern`) instead.
- **`ownership`** — team or `@handle`.
- **`gotchas[]`** — list of traps. Appendable.
- **`decisions[]`** — list of decisions with reasons. Appendable.
- **`invariants[]`** — list of must-be-true statements. Appendable.
- **`required_checks[]`** — list of tests/lints/validations that must pass. Appendable.
- **`tags[]`** — comma-separated labels for dashboard filtering.

### System-managed (don't write directly)

- **`status`** — `draft` | `reviewed` | `superseded` | `archived`.
- **`version`** — bumps on every PATCH.
- **`stale`** / **`stale_reason`** — set automatically when covered code changes on a tracked branch.
- **`where_repos[]`** — every repo where this topic applies. Auto-populated by `context update` (current repo) and explicit `context link`.
- **`references_topics[]`** — slugs parsed from `[[wiki-links]]` in free-text fields.
- **`referenced_by[]`** — computed at read time from other topics' `references_topics`.
- **`history[]`** — audit trail of events.

### Anchors

- **`patterns[]`** — globs (e.g. `src/billing/**`) that claim a slice of the codebase. `--pattern` replaces the whole list; `--add-pattern` / `--remove-pattern` are atomic and idempotent.

## Append vs Replace semantics

This is the most common source of accidental data loss. Read carefully:

| Flag | Semantics |
|---|---|
| `--what "..."` | **Replace** the `what` field |
| `--how "..."` | **Replace** the `how` field |
| `--where "..."` | **Replace** the single `where` path |
| `--ownership "..."` | **Replace** the ownership field |
| `--gotcha "..."` (repeatable) | **Append** a gotcha (never clobbers existing) |
| `--gotchas "..."` | **Replace** the whole gotchas list — destructive |
| `--decision "..."` (repeatable) | **Append** a decision |
| `--decisions "..."` | **Replace** the whole decisions list — destructive |
| `--invariant "..."` (repeatable) | **Append** an invariant |
| `--check "..."` (repeatable) | **Append** a required check |
| `--pattern "..."` (repeatable on `add`) | Set initial patterns |
| `--pattern "..."` on `update` | **Replace** the whole patterns list — destructive |
| `--add-pattern "..."` (repeatable, idempotent) | Append a single pattern (no-op if already there) |
| `--remove-pattern "..."` (repeatable, idempotent) | Remove a single pattern (no-op if absent) |

**Rule of thumb:** singular flags (`--gotchas`, `--decisions`, `--pattern` on update) REPLACE; their singular/append counterparts (`--gotcha`, `--decision`, `--add-pattern`) APPEND. When in doubt, append.

## Batching — one PATCH, not N

Every `context update` bumps `version` and writes a history event. Multiple updates pollute the trail and risk lost appends under concurrency. Batch every related change into ONE invocation:

```bash
# YES — one atomic update, one event, one version bump
driftless context update billing-flow \
  --gotcha "Stripe webhooks deliver out-of-order — idempotency-key required" \
  --gotcha "Refund > 90 days breaks reconciliation" \
  --decision "Async webhook handler chosen over sync to absorb retry storms" \
  --invariant "Webhook handlers must be idempotent on event.id" \
  --check "Replay tests must pass under chaos-mode" \
  --add-pattern "src/billing/webhooks/**" \
  --rel depends_on:stripe-webhook-ingest

# NO — three updates, three events, three version bumps, noisy history
driftless context update billing-flow --gotcha "..."
driftless context update billing-flow --gotcha "..."
driftless context update billing-flow --decision "..."
```

All append flags are repeatable in a single invocation. The API runs the UPDATE under a row lock, so concurrent appends from different agents don't lose data.

## Anchoring discipline — in depth

Anchors decide which files Cloud considers "covered" by this topic. Coverage drives:
- Whether `context get --files <file>` matches the topic to a file.
- Whether a push to the file marks the topic stale.
- Whether PR coverage marks the file as covered by topic context.

### Healthy anchor count

| File count covered | Verdict |
|---|---|
| 5–40 | Healthy — narrow enough to mean something specific |
| 41–99 | Wide — verify the topic is actually one concept, not three |
| 100+ | Trap — almost always over-broad; the topic devolves into a catch-all |

The CLI emits `⚠ over-broad anchor` after any `add` / `update` that crosses 100. **Split into narrower topics rather than suppress it.** That warning is structural feedback designed to catch the trap at the point of friction.

### Multi-anchor over wide-glob

Prefer multiple narrow `--pattern` flags over one broad glob:

```bash
# Good — narrow, multi-anchor, conceptually unified
driftless context add checkout-flow \
  --pattern "src/checkout/**" \
  --pattern "src/cart/**" \
  --pattern "src/payments/intent/**"

# Bad — one wide glob, will match too much
driftless context add ecommerce --pattern "src/**"
```

If you find yourself reaching for `src/**`, the answer is **more topics, not a wider glob**. One topic per concept.

### Managing anchors over time

Use atomic, idempotent ops to evolve an anchor set without clobbering the rest of the topic:

```bash
driftless context update billing-flow \
  --add-pattern "src/billing/refunds/**" \
  --remove-pattern "src/billing/legacy/**"
```

Re-running the same `--add-pattern` or `--remove-pattern` is a no-op (idempotent).

## Linking topics

### Weak references — `[[slug]]` wiki-links

Inside any free-text field (`--what`, `--how`, `--decisions`, `--gotcha`, `--invariant`, `--ownership`), write `[[other-topic-slug]]`. The API parses every mention and stores the slug in `references_topics`. The reverse direction (`referenced_by`) is computed at read time.

```bash
driftless context update refund-flow \
  --gotcha "Same race as [[stripe-webhook-ingest]]" \
  --decision "Idempotency-key derived from charge_id, mirroring [[payment-gateway]]"
```

- Slug grammar: lowercase alphanumerics + hyphens, must start with `[a-z0-9]`. Anything else is ignored.
- Self-references are silently stripped.
- Dead links (slugs that don't resolve) are accepted in writes but never produce a backlink on the other side — they sit silently in `references_topics` until you fix them.

**Link when:**
- A gotcha only makes sense given another topic — link instead of restating.
- A decision is the consequence of another — link.
- Two topics describe parts of the same flow — link both directions.

**Do NOT link when:**
- The slug only sounds vaguely related (wiki links are claims about *real* causal/definitional dependencies).
- The target topic doesn't exist yet (dead links rot silently).

### Strong references — typed relations

Unlike `[[slug]]` mentions, relations are first-class typed edges:

```bash
driftless context update <slug> --rel depends_on:other-topic
driftless context relations <slug>     # see incoming and outgoing
driftless context graph <slug>         # local graph in the CLI
```

Relation types: `relates_to`, `depends_on`, `supersedes`, `blocks`, `implements`, `documents`, `risk_for`.

The dashboard Topic Graph (`/graph`) renders the full workspace knowledge graph with color-coded nodes and relation edges.

## Status lifecycle — governance

A topic is **authoritative only if approved**. The lifecycle is a governed state machine: **`draft → proposed → reviewed → archived`**.

| Status | Meaning |
|---|---|
| `draft` | Not yet team-blessed. `--suggest`-generated topics start here. **Agents must not treat as authoritative** — it's a hint. |
| `proposed` | Submitted for review, awaiting a human's approval. |
| `reviewed` | **Reviewed.** A human approved it; the read response carries `governance.authoritative: true` + `approved_by` / `approved_at`. The agent treats this as truth. |
| `archived` | Retired. Hidden from default `context list`. |
| `orphaned` | Code-driven (its repo was deleted) — orthogonal to the governance states. |

The model is **agents propose, humans approve** — `approve`/`reject`/`merge` require a human identity (an ownerless agent key can propose but never bless):

```bash
driftless context propose <slug>     # draft → proposed
driftless context approve <slug>     # → reviewed (authoritative), seals approved_by
driftless context reject <slug>      # proposed → draft
driftless context archive <slug>     # → archived
```

To change an already-`reviewed` topic, **don't overwrite it — open a topic-PR** (`driftless context pr <slug> --open ...`); a human merges it (applies the change + re-approves). See `references/commands.md`.

## Visibility

Every topic lives in the **workspace** and is visible to all its members. The one exception is a **private draft** (`status=draft` + `is_private`): visible only to its creator until they propose it or clear the private flag. Need true isolation between groups? use a separate workspace. (Approving a proposal into Institutional Context is a workspace **role** — owner/admin; members propose.)

## Topic content body

The `--content` flag accepts either inline markdown or a file path (prefixed with `@`):

```bash
# Inline markdown
driftless context add roadmap-q3 \
  --content "# Q3 Roadmap\n\n## Goals\n- Auth redesign\n- Stripe migration"

# From a file
driftless context add billing-flow \
  --content @.driftless/assets/templates/reference.md
```

**YAML frontmatter in content is parsed automatically** — top-level keys map onto topic fields:

```yaml
---
what: Auth system
how: JWT via Clerk
ownership: "@platform"
---
# Additional markdown body follows the frontmatter
```

This is why every template in `.driftless/assets/templates/` starts with a frontmatter block — the agent fills in `what` / `how` / `ownership` in the YAML and adds detail below.
