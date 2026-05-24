---
name: driftless
description: Driftless is the team's shared context layer for AI coding agents in TypeScript/NestJS/Next.js/React repos. Topics are durable engineering memory (what/how/gotchas/decisions/invariants/checks) persisted in Cloud; an optional repo scan adds a deterministic blast radius (entrypoints, callers, deps, auth guards) without any LLM. It informs and never blocks. Use BEFORE editing code to load team context, AFTER discovering a gotcha or architectural decision to persist it, when RESUMING work to see what drifted around your topics, or BEFORE committing to refresh context for your local diff. Triggers in repos containing .driftless/ or an AGENTS.md that mentions driftless, and on phrases like "starting work", "resuming", "what changed since", "how does X work in this repo", "what breaks if I change this", "what calls this", "is this endpoint protected", "blast radius", "about to push", "found a gotcha", "context get", "context load", "graph file", "driftless sync", "driftless".
license: MIT
metadata:
  author: Driftless
  version: 2.0.0
  homepage: https://driftless.icu
  cli: "@driftless-sh/cli"
---

# Driftless

Cloud is the source of truth. You pull team context before coding and persist what you learn back to Cloud. You do not own topics — the team does.

Driftless is a context + drift-awareness layer. It informs; it never blocks.

## When to use this skill

- Starting or resuming a session in any repo that contains `.driftless/` or has an `AGENTS.md` mentioning Driftless.
- About to edit any file or module — you need the team's context and the file's blast radius before touching code.
- You returned after time away and need to catch up on what the team changed around your topics.
- You discovered a gotcha, a decision, an invariant — something durable that future agents and teammates should know.
- About to commit or push — you want to refresh context that may have drifted, including for your local diff.
- You suspect the context layer itself may be untrustworthy (stale, orphaned, draft-heavy).

## CRITICAL: First move per situation

Before anything else, route on the situation:

| Situation | First command |
|---|---|
| Doctor fails or no topics exist | UC0 — Create your first topic. **Ask before running `driftless init`** (repo scan is optional enrichment) |
| Starting or resuming work on a feature | If the repo is already connected, `driftless sync` — then UC1 |
| About to edit a specific file | `driftless context load --files "<path>" --json` (UC1) |
| Learned something worth keeping | UC2 — `driftless context update <slug>` (or `add` if no topic) |
| About to commit or push | `driftless sync` then `driftless context get --diff` (UC3) |
| Need to know if context itself is trustworthy | `driftless context doctor` |

`driftless sync` pulls Cloud state for the current repo — **stale topics** (a topic whose covered code the team changed on a tracked branch since you last looked), **team PR activity**, and suggested topics pending review. It is the deduped "what drifted around my topics" signal, not a raw event feed.

Drift is **scoped to tracked branches**: a topic only goes stale when its covered code changes on the repo's default branch (auto-detected — main/master/whatever) or an extra branch the team opted into. A push to a throwaway feature branch is recorded but never creates false drift. `driftless branches` shows/sets the tracked set; `sync` prints it as the `tracking:` line.

`driftless context doctor` audits the context layer itself — it flags `stale`, `orphaned` (repo deleted), `draft` (suggested, never confirmed), `docs_pending` and `repo_leak` topics. Run it if `context get` results look wrong or before relying heavily on the context layer.

---

## Use Case 1 — Load context BEFORE editing code

**Trigger:** You are about to modify any module, feature, endpoint, guard, service, or package boundary.

**Steps:**

1. **Sync first** (skip only if you ran it less than ~10 min ago):
   ```bash
   driftless sync
   ```

2. **If you know the slug**, get the topic directly. Otherwise search:
   ```bash
   driftless context get <slug>
   driftless context search <keyword>
   ```

3. **Read the full topic response.** Pay attention to:
   - `what` — what this module does
   - `how` — implementation patterns in use
   - `where` — canonical file paths
   - `gotchas` — what has burned the team before
   - `decisions` — why patterns were chosen; do not override without understanding
   - `invariants` — what must always be true; do not violate
   - `history` — recent `FILE_CHANGED` and `UPDATED` events

   **If returning after time away:** check `history` before assuming your previous understanding holds. Files may have changed. Context may have been updated by a PR or another agent. Do not continue from old assumptions.

4. **About to edit specific files — load context + coverage + graph in one shot:**
   ```bash
   driftless context load --files "src/billing/billing.service.ts" --json
   ```

   Returns, per file: the matching `topics` (direct/indirect), the deterministic `graph_facts`, and a `coverage` block: `{ status, match_specificity, high_risk, recommended_action }`. This is the canonical pre-edit call.

5. **You MUST act on `coverage`** — see `references/coverage-interpretation.md` for the full decision tree. Short version:
   - `status: "reviewed"` + `match_specificity: file|component` → context is authoritative; proceed.
   - `status: "partial"` or `match_specificity: pattern|repo` → loose match; verify against code; update topic if you learn something (UC2).
   - `status: "none"` → no team context; read the code, then create a topic before assuming.
   - `high_risk: true` + weak coverage → document the invariant BEFORE changing behavior.
   - Follow `recommended_action` literally (`document-now` / `add-topic` / `refresh-stale` / `reattach-or-remove` / `review-suggested` / `promote-draft` / `none`).

6. **For one-off blast radius checks** without coverage, use the graph directly:
   ```bash
   driftless graph file src/billing/billing.service.ts
   driftless graph impact --files "src/billing/billing.service.ts"
   ```

   Context tells you the team's intent. The code graph tells you the blast radius — deterministically, no LLM. Key fields: `entrypoints[]` (`method`, `path`, `handler`, `guards`), `upstream`, `downstream`, `contracts`, `global_guards`, and per-component `confidence` + `evidence` (`file:line`).

### CRITICAL: Reading guards on entrypoints

- **NestJS** — `guards` is the name from `@UseGuards(X)` (e.g. `"AuthGuard"`). `APP_GUARD` providers go in `global_guards`.
- **Next.js** — `"clerk-auth"` when `auth()` or `currentUser()` from `@clerk/nextjs/server` is invoked in the file. Call-site `file:line` lives in `metadata.guard_evidence`.
- **`guards: []` AND `global_guards: []`** → treat as **"no auth detected — verify"**. **NEVER** assume "public". The repo may use a composed/custom auth decorator or non-Clerk auth helper the scanner can't see. Open the file and verify.

---

## Use Case 2 — Persist what you learned

**Trigger:** You learned something durable — a gotcha, a decision, a constraint, an invariant — that future agents or teammates should know.

**Rule:** If it would have saved you time to know this upfront, save it now. Do NOT keep discoveries in conversation memory only. Cloud is the source of truth.

### If a topic exists for the area

Batch every related change into ONE update — each PATCH bumps version and writes a history event, so N small updates pollute the trail:

```bash
driftless context update billing-flow \
  --gotcha "Stripe webhooks deliver out-of-order — idempotency-key required" \
  --gotcha "Refund > 90 days breaks reconciliation" \
  --decision "Async webhook handler chosen over sync to absorb retry storms" \
  --invariant "Webhook handlers must be idempotent — never assume single delivery" \
  --check "Run integration suite under chaos-mode replay before merging" \
  --add-pattern "src/billing/webhooks/**" \
  --rel depends_on:stripe-webhook-ingest \
  --tags billing,webhooks
```

Append flags (`--gotcha`, `--decision`, `--invariant`, `--check`) are repeatable and additive — every value lands atomically under a row lock, so concurrent agents don't lose appends.

### If no topic covers this area

Use the matching template from `.driftless/assets/templates/` as the body. Each `--kind` has its own idiom — don't invent the structure.

```bash
# Code-context topic — what/how/where + gotchas/decisions
driftless context add billing-flow \
  --kind code-context \
  --content @.driftless/assets/templates/code-context.md \
  --pattern "src/billing/**" --pattern "src/checkout/**" \
  --tags billing

# Architecture decision (ADR) — context/options/decision/consequences
driftless context add async-webhook-handler \
  --kind decision \
  --content @.driftless/assets/templates/decision.md

# Runbook — trigger/symptoms/diagnosis/resolution/rollback
driftless context add stripe-webhook-replay \
  --kind runbook \
  --content @.driftless/assets/templates/runbook.md

# Integration note — vendor/auth/quirks/failure modes
driftless context add stripe-integration \
  --kind integration-note \
  --content @.driftless/assets/templates/integration-note.md
```

After creating, edit the topic in Cloud (dashboard) or via `context update` to replace placeholders with real content.

### Cross-repo

A topic can span multiple repos (`where_repos`). When the same concept lives in another repo you are working in, register it explicitly:

```bash
driftless context link <slug>
```

`context link` registers the repo you are currently in into the topic's `where_repos` and **changes nothing else**. After linking, `context get <slug>` shows `used in (N repos): …` and groups components by repo.

### CRITICAL: Anchoring discipline

Patterns claim a slice of the codebase. Size matters:

| Component count | Verdict |
|---|---|
| 5–40 | Healthy — narrow enough to mean something specific |
| 41–99 | Wide — verify it's truly one concept, not three |
| 100+ | Trap — almost always over-broad; the topic devolves into a catch-all |

The CLI emits `⚠ over-broad anchor` after `add` / `update` when the topic crosses 100. **Don't suppress it — split into narrower topics.**

**Good** (narrow, multi-anchor):
```bash
driftless context add checkout-flow \
  --pattern "src/checkout/**" --pattern "src/cart/**"
```

**Bad** (catch-all that will rot):
```bash
driftless context add backend --pattern "src/**"               # don't
driftless context add all-controllers --pattern "**/*.controller.ts"  # don't
```

If you reach for `src/**`, the answer is more topics, not a wider glob. One topic per *concept*.

Manage patterns atomically — `--add-pattern` / `--remove-pattern` are repeatable AND idempotent:

```bash
driftless context update billing-flow \
  --add-pattern "src/billing/refunds/**" \
  --remove-pattern "src/billing/legacy/**"
```

See `references/topic-anatomy.md` for the full reference on kinds, fields, append-vs-replace semantics, and status lifecycle.

### Linking topics with `[[slug]]`

Topics are not islands. When the thing you are documenting only makes sense in the context of *another* topic — that's a link, not a copy-paste of duplicated detail.

Inside any free-text field (`--what`, `--how`, `--decisions`, `--gotcha`, `--invariant`, `--ownership`), write `[[other-topic-slug]]`. The API parses every mention and stores the slug in `references_topics`; the other topic then sees this one in its `referenced_by` list.

```bash
driftless context update refund-flow \
  --gotcha "Refund webhooks arrive out-of-order — same race as [[stripe-webhook-ingest]]" \
  --decision "Idempotency-key derived from charge_id, mirroring [[payment-gateway]]"
```

**Link when:** a gotcha only makes sense given another topic; a decision is the consequence of another; two topics describe parts of the same flow.

**Do NOT link when:** the slug only sounds vaguely related; or the target topic doesn't exist yet (dead links never produce a backlink — the trace rots silently).

### Typed semantic relations

Unlike `[[slug]]` mentions (weak backlinks), relations are strong typed edges:

```bash
driftless context update <slug> --rel depends_on:other-topic
driftless context relations <slug>     # see incoming and outgoing
driftless context graph <slug>         # local graph around this topic
```

Relation types: `relates_to`, `depends_on`, `supersedes`, `blocks`, `implements`, `documents`, `risk_for`.

---

## Use Case 3 — First-time setup (topic-first)

**Trigger:** `driftless doctor` fails, or `driftless context list` returns empty.

Driftless does **not** require a repo scan to start. You begin by creating topics — durable engineering memory — without running `driftless init`. Repo scanning is **optional enrichment** that adds code anchors, component discovery, graph coverage, impact analysis, and stale-context detection.

### Steps (topic-first, ~60 seconds)

1. **Authenticate:**
   ```bash
   driftless login
   # non-interactive: driftless login --key <api-key>
   ```

2. **Install the agent skill** — writes `CLAUDE.md`, `AGENTS.md`, and `.driftless/` (skill body + references + templates):
   ```bash
   driftless install-skill
   ```

3. **Create your first topic** (no scan required):
   ```bash
   driftless context add onboarding-context \
     --kind domain-map \
     --what "Shared context map for this repo or workflow." \
     --how "Captures what humans and agents need to know before working here."
   ```

4. **Fill in structured context** as you learn:
   ```bash
   driftless context update onboarding-context \
     --gotcha "What I learned that was not documented" \
     --decision "Why the team does it this way" \
     --invariant "What must always be true" \
     --check "What to verify before changing this area"
   ```

5. **Anchor to file paths when ready** (still optional — many topics never need anchors):
   ```bash
   driftless context update onboarding-context \
     --pattern "apps/*/src/**" --pattern "docs/**"
   ```

### Repo scan — optional enrichment

`driftless init` runs the scanner: it adds the component baseline, code anchors, coverage, the deterministic graph, and stale-context detection. **Ask the user before running it.** Driftless is useful without it.

```bash
driftless init
driftless init --src apps/api/src                  # monorepo with sources in a subfolder
driftless init --src apps/api/src --suggest        # also create draft topics for each module
```

Framework is auto-detected from `package.json` (NestJS, Next.js, standalone React). Use `--framework` only to override. `--suggest` produces draft topics for each module — you fill them in and promote to `reviewed` via UC2.

### Verify setup

```bash
driftless doctor
```

All checks should be `ok` or `warn`. If you ran `--suggest`:

```bash
driftless context list --suggested
```

For each suggested topic, confirm it, fill in content, and promote to `reviewed` (UC2).

### If `init` reports existing docs

Ask the user which topic each doc belongs to before syncing:

> "I found 3 docs (README.md, docs/auth.md, docs/billing.md). Which context topic should each one be linked to? I'll sync them once you confirm."

Then:
```bash
driftless context sync <slug> --doc path/to/doc.md
```

---

## Use Case 4 — Pre-commit refresh

**Trigger:** About to commit, push, or wrap up the task.

```bash
driftless sync                          # catch up on what the team pushed while you worked
driftless context get --diff            # topics matching your local uncommitted changes
```

If `--diff` surfaces a stale topic that your change touches, update it (UC2) BEFORE pushing. The point of pre-commit is to leave the context layer healthier than you found it.

---

## Examples

### Example 1 — Starting work on a refund flow
**User says:** "I'm going to build the refund flow on top of the existing billing module."

**Actions:**
```bash
driftless sync
driftless context search refund                      # → no match
driftless context get billing                        # → existing topic, read decisions + gotchas
driftless context load --files "src/billing/refunds/refund.service.ts,src/billing/refunds/refund.controller.ts" --json
# coverage shows status:"partial", recommended_action:"add-topic"
driftless graph impact --files "src/billing/refunds/refund.service.ts"
# implement…
driftless context add refund-flow \
  --kind code-context \
  --content @.driftless/assets/templates/code-context.md \
  --pattern "src/billing/refunds/**"
```

### Example 2 — Webhooks delivering twice
**User says:** "Stripe webhooks are arriving twice in production. Help me debug."

**Actions:**
```bash
driftless context get billing                        # → gotcha about idempotency? if yes, follow it
# if no gotcha exists about it but the fix is known, persist after:
driftless context update billing \
  --gotcha "Stripe delivers webhooks at-least-once; consumers MUST be idempotent on event.id" \
  --invariant "Webhook handlers must read event.id and short-circuit on duplicate" \
  --check "Replay tests must pass with duplicate event.id in chaos suite"
```

### Example 3 — First-time setup in a fresh repo
**User says:** "Set up Driftless for this repo."

**Actions:**
```bash
driftless login                                       # if not already
driftless install-skill                               # writes CLAUDE.md, AGENTS.md, .driftless/
driftless context add onboarding-context \
  --kind domain-map \
  --what "Shared context map for this repo." \
  --how "Captures what humans and agents need to know before working here."
# ASK before running driftless init — scan is optional
```

### Example 4 — Pre-commit refresh
**User says:** "I'm about to push. Anything I should refresh first?"

**Actions:**
```bash
driftless sync                                        # stale topics? team PRs?
driftless context get --diff                          # topics matching local diff
# If any topic is stale and your change touched it, update before pushing:
driftless context update <slug> --gotcha "…" --decision "…"
git push
```

### Example 5 — Coverage looks suspicious
**User says:** "`context get` is returning weird results, can I trust it?"

**Actions:**
```bash
driftless context doctor
# Inspect categories: stale / orphaned / draft / docs_pending / repo_leak
# Fix orphaned and repo_leak before relying on context.
```

---

## Troubleshooting

For the full catalog, see `references/troubleshooting.md`.

**`driftless: command not found`** → `npm install -g @driftless-sh/cli`

**`Unauthorized` / `403`** → `driftless login --key <api-key>` (get key at driftless.icu → Settings → API Keys)

**`context get` returns stale content** → topic is marked stale (files it tracks changed). Read `stale_reason`. After reviewing the current code, update the topic (UC2).

**`context load` returns no topics for a file you expect to be covered** → no topic anchors that path. Run `context list` to see what exists, then either add a pattern via `context update <slug> --add-pattern "<glob>"` or create a new topic (UC2).

**`sync` prints nothing under "stale"** → either no drift since you last looked (healthy), or the GitHub App is not installed (Cloud has no push events to compute drift from). Check `driftless doctor` — the GitHub App line is `INFO` when not installed; install at driftless.icu → Settings → Integrations.

**`graph file` returns empty entrypoints / upstream** → file may be standalone (no routes reach it) or the scanner could not parse it (low confidence on the file's components). Open the file and verify the auth surface manually before assuming "public".

---

## Performance Notes

- **Take your time to read `coverage.status` before treating context as authoritative.** A `partial` topic is a hint, not a contract. Verify against code.
- **Quality of a topic matters more than speed.** Narrow anchors, batched updates, real causal `[[links]]`. A noisy topic rots; a precise one compounds.
- **Do NOT skip `driftless sync`.** Drift detection is the safety net — if the team changed something around your topic, you want to know before you write code.
- **Do NOT assume "no guards detected" means "public".** Composed decorators and custom auth helpers exist. `guards: []` AND `global_guards: []` means **verify in code**.
- **Do NOT run `driftless init` without user permission.** The repo scan is optional enrichment. Topics work without it.
- **Do NOT split a discovery into N updates.** Every PATCH bumps version and writes a history event. Batch related `--gotcha` / `--decision` / `--add-pattern` into one invocation.
- **Do NOT create catch-all topics** (`--pattern "src/**"`). One topic per concept; multi-anchor instead.

---

## See also

- `references/commands.md` — Full CLI command reference
- `references/workflow.md` — The Driftless Loop in depth
- `references/cloud.md` — Cloud data model and architecture
- `references/topic-anatomy.md` — Kinds, fields, append-vs-replace, linking
- `references/coverage-interpretation.md` — Coverage status decision tree
- `references/troubleshooting.md` — Error / Cause / Solution catalog

Common flags available on most context commands:
- `--dry-run` — preview changes without writing
- `--json` — machine-readable output
- `--status <reviewed|draft>` — set topic status on create or update
