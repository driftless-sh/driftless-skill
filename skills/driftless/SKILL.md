---
name: driftless
description: Driftless is the team's shared context layer for AI coding agents in any repo. Topics are durable engineering memory persisted in Cloud — what / how / gotchas / decisions / invariants / checks — anchored to globs that point at real files. The PR bot reads each merged change against the topic layer and surfaces coverage gaps directly in the PR comment, so the team learns what to document next. It informs and never blocks. Use BEFORE editing code to pull team context (`context get` / `context search`), AFTER discovering a gotcha or architectural decision to persist it (`context update` / `context add`), when RESUMING work to see what drifted around your topics (`sync`), and BEFORE committing to refresh context for your local diff (`context get --diff`). Triggers in repos containing `.driftless/` or an `AGENTS.md` mentioning Driftless, and on phrases like "starting work", "resuming", "what changed since", "how does X work in this repo", "found a gotcha", "about to push", "context get", "context add", "context update", "driftless sync", "driftless".
license: MIT
metadata:
  author: Driftless
  version: 3.0.0
  homepage: https://driftless.icu
  cli: "@driftless-sh/cli"
---

# Driftless

Cloud is the source of truth. You pull team context before coding and persist what you learn back to Cloud. You do not own topics — the team does.

Driftless is a context + drift-awareness layer. It informs; it never blocks. Topics are anchored to globs, the PR bot reports coverage on every PR, and gaps surface as nudges in the comment rather than as out-of-band reports.

## When to use this skill

- Starting or resuming a session in any repo that contains `.driftless/` or has an `AGENTS.md` mentioning Driftless.
- About to edit any file or module — you need the team's prior context (what / how / gotchas / decisions / invariants) before touching code.
- You returned after time away and need to catch up on what the team changed around your topics.
- You discovered a gotcha, a decision, an invariant — something durable that future agents and teammates should know.
- About to commit or push — you want to refresh context for your local diff and update anything that drifted.
- You suspect the context layer itself may be untrustworthy (stale, orphaned, zombie, draft-heavy).

## CRITICAL: First move per situation

Before anything else, route on the situation:

| Situation | First command |
|---|---|
| Empty workspace, no topics yet | UC0 — `driftless login` then `driftless install-skill` then create your first topic |
| Starting or resuming work on a feature | `driftless sync` then `driftless context search <kw>` / `context get <slug>` (UC1) |
| About to edit a specific file | `driftless context get --files "<path>"` (UC1) |
| Learned something worth keeping | UC2 — `driftless context update <slug>` (or `add` if no topic) |
| About to commit or push | `driftless sync` then `driftless context get --diff` (UC3) |
| Need to know if context itself is trustworthy | `driftless context doctor` |

`driftless sync` pulls Cloud state for the current repo — **stale topics** (a topic whose covered code the team changed on a tracked branch since you last looked), **team PR activity**, and suggested topics pending review. It is the deduped "what drifted around my topics" signal, not a raw event feed.

Drift is **scoped to tracked branches**: a topic only goes stale when its covered code changes on the repo's default branch (auto-detected — main/master/whatever) or an extra branch the team opted into. A push to a throwaway feature branch is recorded but never creates false drift. `driftless branches` shows/sets the tracked set; `sync` prints it as the `tracking:` line.

`driftless context doctor` audits the context layer itself — it flags `stale`, `orphaned` (repo deleted), `zombie` (anchor patterns no longer match any file in this checkout), `draft` (suggested, never confirmed), `docs_pending`, and `repo_leak` topics. Run it if `context get` results look wrong or before relying heavily on the context layer.

---

## Use Case 1 — Retrieve context BEFORE editing code

**Trigger:** You are about to modify any module, feature, endpoint, service, or package boundary.

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

3. **About to edit specific files — match topics by path:**
   ```bash
   driftless context get --files "src/billing/billing.service.ts,src/billing/refunds/*"
   ```
   Returns every topic whose anchors (`patterns` or `where_files`) match any of the given paths. Use this when you know the file but not the topic name.

4. **Read the full topic response.** Pay attention to:
   - `what` — what this module does
   - `how` — implementation patterns in use
   - `where` — canonical file paths and glob anchors
   - `gotchas` — what has burned the team before
   - `decisions` — why patterns were chosen; do not override without understanding
   - `invariants` — what must always be true; do not violate
   - `history` — recent `FILE_CHANGED` and `UPDATED` events

   **If returning after time away:** check `history` before assuming your previous understanding holds. Files may have changed. Context may have been updated by a PR or another agent. Do not continue from old assumptions.

5. **No topic matches the file you are about to edit?**
   Read the code first, then create a topic before assuming. Driftless will only ever know what the team writes down — the absence of a topic is a gap to close (UC2), not permission to skip the documentation step.

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

**Pattern validation at write time.** Every `--pattern` and `--add-pattern` is globbed against your local checkout before the API call. The CLI prints, per pattern:

- `✓ pattern "<glob>" matches N files — e.g. <samples>` when the glob lands on real source
- `⚠ over-broad anchor` when one pattern matches more than 100 files
- `⚠ pattern "<glob>" matches 0 files — likely unintended` when every match falls under `node_modules/`, `dist/`, `.git/` etc
- **Blocking error** when a pattern matches 0 real files. The create/update is aborted; fix the glob or use `--where` for an explicit path

Trust the validator. If it says 0 matches, the topic would have been instantly stale on day one.

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

`context link` registers the repo you are currently in into the topic's `where_repos` and **changes nothing else**. After linking, `context get <slug>` shows `used in (N repos): …` and groups files by repo.

### CRITICAL: Anchoring discipline

Patterns claim a slice of the codebase. Size matters:

| Match count | Verdict |
|---|---|
| 5–40 | Healthy — narrow enough to mean something specific |
| 41–99 | Wide — verify it's truly one concept, not three |
| 100+ | Trap — almost always over-broad; the topic devolves into a catch-all |

The CLI emits `⚠ over-broad anchor` after `add` / `update` when a pattern crosses 100 matches. **Don't suppress it — split into narrower topics.**

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
driftless context graph <slug>         # local relation graph around this topic
```

Relation types: `relates_to`, `depends_on`, `supersedes`, `blocks`, `implements`, `documents`, `risk_for`.

---

## Use Case 0 — First-time setup

**Trigger:** `driftless context list` returns empty, or you just ran `driftless install-skill` for the first time.

Driftless is a notetaker, not an indexer. Onboarding is three commands: authenticate, install the skill, create the first topic.

### Steps (~60 seconds)

1. **Authenticate:**
   ```bash
   driftless login
   # non-interactive: driftless login --key <api-key>
   ```

2. **Install the agent skill** — writes `CLAUDE.md`, `AGENTS.md`, and `.driftless/` (skill body + references + templates):
   ```bash
   driftless install-skill
   ```

3. **Create your first topic**:
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

5. **Anchor to file paths once you know which slice this topic covers** (the CLI validates the glob locally before writing):
   ```bash
   driftless context update onboarding-context \
     --add-pattern "apps/*/src/**" --add-pattern "docs/**"
   ```

### Verify setup

```bash
driftless doctor
driftless context doctor
```

`driftless doctor` checks auth, connectivity, workspace, repo link, and AGENTS.md presence. `driftless context doctor` audits the topic layer (stale / orphaned / zombie / draft / docs_pending / repo_leak).

### How topics grow over time

You don't need to document everything upfront. The PR bot does the prompting:

- Every PR posts a comment summarising **which topics it touched** and **how many files in the PR were not covered by any topic**.
- For the uncovered files, the bot suggests the next step inline: `context update <slug> --add-pattern "<glob>"` to extend an existing topic, or `context add <slug> --pattern "<glob>"` to create a new one.

That feedback loop is the engine of growth.

---

## Use Case 3 — Pre-commit refresh

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
driftless context get --files "src/billing/refunds/refund.service.ts,src/billing/refunds/refund.controller.ts"
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

### Example 5 — Context looks suspicious
**User says:** "`context get` is returning weird results, can I trust it?"

**Actions:**
```bash
driftless context doctor
# Inspect categories: stale / orphaned / zombie / draft / docs_pending / repo_leak
# Resolve orphaned, repo_leak, and zombie first — they indicate broken anchors.
# For zombies: context update <slug> --remove-pattern "<old>" --add-pattern "<new>" — or archive if obsolete.
```

### Example 6 — PR bot surfaces a gap
**User says (in PR review):** "The Driftless comment says 6 files in this PR aren't covered by any topic."

**Actions:**
```bash
# Look at the listed paths in the comment's "files not anchored to any topic" section.
# If they belong to an existing topic:
driftless context update <slug> --add-pattern "<glob covering those paths>"
# If they form a new concept:
driftless context add <slug> \
  --kind code-context \
  --content @.driftless/assets/templates/code-context.md \
  --pattern "<glob covering those paths>"
# The next PR touching the same area will show those files as covered.
```

---

## Troubleshooting

For the full catalog, see `references/troubleshooting.md`.

**`driftless: command not found`** → `npm install -g @driftless-sh/cli`

**`Unauthorized` / `403`** → `driftless login --key <api-key>` (get key at driftless.icu → Settings → API Keys)

**`context get` returns stale content** → topic is marked stale (files it tracks changed). Read `stale_reason`. After reviewing the current code, update the topic (UC2).

**`context get --files "..."` returns no topics for a file you expect to be covered** → no topic anchors that path. Run `context list` to see what exists, then either add a pattern via `context update <slug> --add-pattern "<glob>"` or create a new topic (UC2). Alternatively, run `driftless context doctor` and look for `zombie` — the anchor may have rotted.

**`context add` errors with "pattern matches 0 files"** → the glob is wrong or the path doesn't exist in this checkout. Verify against the repo root, broaden the glob (e.g. `src/billing/**` instead of `src/billing/*.ts`), or use `--where` for an explicit path.

**`sync` prints nothing under "stale"** → either no drift since you last looked (healthy), or the GitHub App is not installed (Cloud has no push events to compute drift from). Check `driftless doctor` — the GitHub App line is `INFO` when not installed; install at driftless.icu → Settings → Integrations.

**PR comment never appears on new PRs** → the GitHub App is not installed on the repo, or the repo is not linked to the workspace. Run `driftless doctor` to confirm. Install the App at driftless.icu → Settings → Integrations.

---

## Performance Notes

- **Quality of a topic matters more than speed.** Narrow anchors, batched updates, real causal `[[links]]`. A noisy topic rots; a precise one compounds.
- **Trust the pattern validator.** When the CLI says a glob matches 0 real files, do not force it — the topic will be a zombie before it is useful.
- **Do NOT skip `driftless sync`.** Drift detection is the safety net — if the team changed something around your topic, you want to know before you write code.
- **Do NOT split a discovery into N updates.** Every PATCH bumps version and writes a history event. Batch related `--gotcha` / `--decision` / `--add-pattern` into one invocation.
- **Do NOT create catch-all topics** (`--pattern "src/**"`). One topic per concept; multi-anchor instead.
- **Use the PR comment as your gap-finder.** When the bot lists uncovered files, that is the precise place to add a pattern or a new topic.

---

## See also

- `references/commands.md` — Full CLI command reference
- `references/workflow.md` — The Driftless Loop in depth
- `references/cloud.md` — Cloud data model and architecture
- `references/topic-anatomy.md` — Kinds, fields, append-vs-replace, linking
- `references/troubleshooting.md` — Error / Cause / Solution catalog

Common flags available on most context commands:
- `--dry-run` — preview changes without writing
- `--json` — machine-readable output
- `--status <reviewed|draft>` — set topic status on create or update
