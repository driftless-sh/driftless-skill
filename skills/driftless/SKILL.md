---
name: driftless
description: Manages shared repo context for AI coding sessions in Driftless-enabled repositories. Loads team knowledge before coding, catches you up on what drifted around your topics when you resume (a topic whose covered code the team changed), persists discoveries to Cloud, and gives a deterministic blast radius for any file. Driftless is a context + drift-awareness layer, not a linter or gate — it never blocks you. Use when starting or resuming work in a repo with Driftless, about to modify any module or feature, returning after time away and need to catch up on what changed, discovered a gotcha or architectural decision worth saving, or before pushing to refresh context that drifted. Also use before editing a specific file to understand its blast radius (entrypoints, callers, dependencies, auth guards) deterministically. Triggers on: "starting work", "resuming", "what changed since", "how does X work in this repo", "what breaks if I change this", "what calls this", "is this endpoint protected", "blast radius", "about to push", "found a gotcha", "context get", "graph file", "driftless".
---

# Driftless

Cloud is the source of truth. You pull context before coding and persist what you learn back to Cloud. You do not own topics — the team does.

## CRITICAL: Detect your situation first

Before anything else, run:

```bash
driftless doctor
```

Route based on result:

| Situation | What to do |
|---|---|
| Doctor fails or no topics exist | → UC0: First-time setup |
| Starting or resuming work on a feature | → Run `driftless sync`, then UC1 |
| About to edit a specific file | → UC1, then `driftless graph file <path>` |
| Learned something worth keeping | → UC2: Save discovery |
| About to commit or push | → Run `driftless sync`, then `driftless context get --diff` and update stale context |
| Need to know if context itself is trustworthy | → `driftless context doctor` |

`driftless sync` is your default starting command. It pulls Cloud state for the current repo — **stale topics** (a topic whose covered code the team changed since you last looked), recent **team PR activity**, and suggested topics pending review. It is the deduped "what drifted around my topics" signal, not a raw event feed. Run it before touching any code.

Drift is **scoped to tracked branches**: a topic only goes stale when its covered code changes on the repo's default branch (auto-detected — main/master/whatever) or an extra branch the team opted into. A push to a throwaway feature branch is recorded but never creates false drift. `driftless branches` shows/sets the tracked set; `sync` prints it as the `tracking:` line so the scope is never invisible.

`driftless context doctor` audits the context layer itself — it flags stale, orphaned (repo deleted), draft (suggested, never confirmed), docs-pending and repo-leak topics. Run it if `context get` results look wrong or before relying heavily on the context layer.

---

## UC0 — First-time setup

**When:** `driftless doctor` fails, or `driftless context list` returns empty.

### Steps

**1. Initialize the repo:**

```bash
driftless init
```

This scans the repo, uploads the baseline, and installs the AGENTS.md skill. Takes ~30 seconds. It will report how many existing docs it found — those are NOT auto-synced.

**Monorepo?** If your code lives in a subfolder (e.g. `apps/api/src`), pass `--src`:

```bash
driftless init --src apps/api/src
```

Git root stays at cwd; scanner starts from the given path.

**Want suggested context topics?** Add `--suggest`:

```bash
driftless init --suggest
# or both:
driftless init --src apps/api/src --suggest
```

Without `--suggest`, no topics are created — you add them in UC2. Suggested
topics are drafts until a human or agent fills in `what`/`how` and promotes them.

**2. Verify setup:**

```bash
driftless doctor
```

All checks should be `ok` or `warn`. Fix any `fail` entries before proceeding.

If you ran `--suggest`, view the auto-generated topics:

```bash
driftless context list --suggested
```

For each suggested topic, confirm it, fill in `what`/`how`, and update it (UC2). That removes the suggestion flag and makes it real team context.

**3. Ask the user about their docs:**

If `init` reported N docs found, ask the user which module each doc belongs to before syncing:

> "I found 3 docs (README.md, docs/auth.md, docs/billing.md). Which context topic should each one be linked to? I'll sync them once you confirm."

Then sync each one:

```bash
driftless context sync <slug> --file path/to/doc.md
```

**4. Proceed to UC1** to load context for the area you're about to work on.

### If init fails

```bash
# Not authenticated
driftless login --key <api-key>
# Get your key at: driftless.icu → Settings → API Keys

# Then retry
driftless init
```

---

## UC1 — Load context before touching code

**When:** Starting or resuming work on any module or feature. Run this before writing a single line.

### Steps

**1. Find the relevant topic:**

```bash
# If you know the slug
driftless context get <slug>

# If you don't know the slug
driftless context search <topic>
```

Example:

```bash
driftless context search "payment"
→ Found: billing, payment-gateway, stripe-adapter

driftless context get billing
```

**2. Read the full response. Pay attention to:**

- `what` — what this module does
- `how` — implementation patterns in use
- `where` — canonical file paths
- `gotchas` — what has burned the team before
- `decisions` — why patterns were chosen; do not override without understanding
- `history` — recent `FILE_CHANGED` and `UPDATED` events

**CRITICAL — If returning after time away:** Check `history` before assuming your previous understanding is still valid. Files may have changed. Context may have been updated by a PR or another agent. Do not continue from old assumptions without reviewing recent events.

**3. Repeat for every module you plan to touch.** One `context get` per area.

**4. Before editing a file, get its deterministic code graph:**

Context tells you the team's intent. The code graph tells you the blast radius — deterministically, no LLM.

```bash
# What is around this file: entrypoints (routes that reach it), upstream
# callers, downstream deps, contracts (DTOs), and auth guards.
driftless graph file src/billing/billing.service.ts

# What can a change here break — the upstream consumers and the HTTP
# routes that become reachable-and-breakable:
driftless graph impact --files "src/billing/billing.service.ts"
```

Read it as the answer to **"what do I need to know before I touch this, and what breaks if I get it wrong"**:

- `entrypoints` — the HTTP routes that reach this code, each with its `guards`. If `guards` is empty AND `global_guards` is empty, treat it as **`no auth detected — verify`**, NOT as "public". The repo may use a global APP_GUARD or a composed auth decorator the scanner can't see.
- `upstream` / `downstream` — callers vs. dependencies.
- `contracts` — the DTOs/inputs this path accepts.
- Every fact carries `confidence` and `evidence` (`file:line`). Low confidence → open the file and verify; do not trust blindly.

Flags: `--json` (machine-readable, for agents), `--depth N` (1–6, default 3), `--direction consumers|dependencies|both` (impact only; default `consumers` = blast radius).

`graph impact` is also the right pre-edit check for risky changes and the input to a careful PR.

**5. About to edit specific files — load context + coverage in one shot:**

```bash
driftless context load --files "src/billing/billing.service.ts" --json
```

Returns, per file: the relevant `topics` (direct/indirect), the deterministic `graph_facts`, and a `coverage` block: `{ status, high_risk, recommended_action }`. This tells you **whether the loaded context can actually be trusted for this file before you touch it.**

**You MUST act on `coverage`, not just read it:**

- `status: "reviewed"` with `match_specificity: "file"` → context is authoritative for this file. Proceed.
- `status: "partial"` or `match_specificity` of `pattern`/`repo`/`weak` → a topic only *loosely* covers this file (broad pattern, not this exact file). Do **NOT** treat it as authoritative — verify against the actual code; if you learn something durable, update the topic (UC2).
- `status: "none"` → **no context covers this file.** Do not assume an unrelated topic applies. After you understand the code, create/anchor a topic (UC2).
- `high_risk: true` and not strongly covered → auth/sensitive surface without solid context. Verify in code and document the invariant (UC2) **before** changing behavior.
- Follow `recommended_action` literally:
  - `create_or_link_reviewed_topic` / `create_or_link_topic` → UC2: add a topic for this area.
  - `refresh_stale_topic` → UC2: update the stale topic against current code.
  - `reattach_or_remove_topic` → the covering topic is orphaned; flag it for reattach/removal, do not rely on it.
  - `review_suggested_topic` / `promote_draft_topic` → confirm/flesh out the topic before trusting it.
  - `ok` → coverage is solid; proceed.

Only `status: "reviewed"` + strong (`file`) coverage means "the loaded context is authoritative." Anything else means **verify against code, then re-anchor (UC2)** — this is how context stays true to the repo over time, not just at init.

### If no topic exists for your area

```bash
driftless context list    # see everything available
```

Work from the closest available topic. After you understand the area, create a topic in UC2.

---

## UC2 — Save what you discovered

**When:** You learned something durable — a gotcha, a decision, a pattern — that future agents or teammates should know. Do not keep discoveries in conversation memory only. Cloud is the source of truth.

### Rule

**If it would have saved you time to know this upfront, save it now.**

### Steps

**If the topic already exists:**

```bash
driftless context update <slug> \
  --gotcha "What I learned that wasn't documented" \
  --gotcha "Another gotcha if needed" \
  --decisions "Why the team does it this way"
```

You can pass `--gotcha` multiple times — all values are appended without overwriting existing ones.

**Cross-repo:** a topic can span multiple repos (`where_repos`). When the same concept lives in another repo you are working in, record that reference explicitly:

```bash
driftless context link <slug>
```

`context link` registers the repo you are currently in into the topic's `where_repos` and **changes nothing else** (no what/how/gotchas/decisions). It is the clean way to say "this topic is also used here". After linking, `context get <slug>` shows `used in (N repos): …` and groups components by repo. (`context update` also auto-links the repo you run it from as a side effect — but use `context link` when linking is all you want, instead of inventing a throwaway update.)

**If no topic exists for this area:**

```bash
driftless context add "<slug>" \
  --what "What this module does" \
  --how "How it is implemented" \
  --where "src/path/to/module/"
```

**If syncing an existing file to a topic:**

```bash
driftless context sync <slug> --file path/to/file.md
```

### What is worth saving

- A constraint not obvious from reading the code
- Why a pattern was chosen over a simpler alternative
- An auth or guard requirement not yet documented as an invariant
- Which files are quarantined and why
- A bug you encountered and how it manifests

### What is NOT worth saving

- Notes relevant only to your current task
- Things clearly readable from the code itself
- Temporary workarounds you have already removed

### Anchoring discipline

Patterns (`--pattern`, `--add-pattern`) are how a topic claims its slice of the codebase. The size of that slice decides whether the topic is useful or noise.

**Heuristic — what a healthy topic looks like:**

| Component count | Verdict |
|---|---|
| 5–40 | Healthy — narrow enough to mean something specific |
| 41–99 | Wide — verify the topic is actually one concept, not three |
| 100+ | Trap — almost always over-broad; the topic devolves into a catch-all and `context get` returns generic guidance instead of the precise thing you needed |

The CLI emits a non-blocking `⚠ over-broad anchor` warning after any `context add` / `context update` that crosses the 100 threshold. Heed it.

**Good — narrow, conceptual, multi-anchor:**

```bash
driftless context add "checkout-flow" \
  --pattern "src/checkout/**" \
  --pattern "src/cart/**" \
  --what "Cart → checkout → payment confirmation"
```

```bash
driftless context update auth-boundaries \
  --add-pattern "src/auth/refresh/**" \
  --add-pattern "src/auth/oauth/**"
```

**Bad — catch-all that will rot:**

```bash
driftless context add "backend" --pattern "src/**"      # don't
driftless context add "all-controllers" --pattern "**/*.controller.ts"  # don't
```

If you find yourself reaching for `src/**`, the answer is more topics, not a wider glob. One topic per *concept*, anchored to the directories that concept actually lives in.

**Batch appends — one PATCH, not N:**

Every `context update` bumps version and writes a history event. Batch everything you learned into ONE invocation, never split:

```bash
# YES — one atomic update, one event, one version bump
driftless context update billing-flow \
  --gotcha "Stripe webhooks deliver out-of-order — idempotency-key required" \
  --gotcha "Refund > 90 days breaks reconciliation" \
  --decision "Async webhook handler chosen over sync to absorb retry storms" \
  --add-pattern "src/billing/webhooks/**"

# NO — three updates, three events, three version bumps, noisy history
driftless context update billing-flow --gotcha "..."
driftless context update billing-flow --gotcha "..."
driftless context update billing-flow --decision "..."
```

**Removing a stale anchor:**

```bash
driftless context update billing-flow --remove-pattern "src/billing/legacy/**"
```

Idempotent: running the same `--remove-pattern` twice is a no-op the second time.

---

## Common issues

**`driftless: command not found`**

```bash
npm install -g @driftless-sh/cli
```

**`Unauthorized` or `403`**

```bash
driftless login --key <api-key>
```

**`No topics found`**

Run `driftless init` from the repo root. Driftless has not been initialized for this repo.

**`context get` returns stale content**

The topic is marked stale — files it tracks changed since the last update. Read `stale_reason` in the output. After reviewing the current code, update the topic (UC2).

**Need context for local changes**

Use `driftless context get --diff` to load topics matching the current local diff, then update stale or missing context with UC2.

---

## See also

- [Full command reference](references/commands.md)
- [Detailed workflow guide](references/workflow.md)
