---
name: driftless
description: Driftless is the team's shared context layer for AI coding agents in any repo. Topics are durable engineering memory in Cloud — what / how / gotchas / decisions / invariants — anchored to globs over real files. Core loop: persist what you learn after a session, pull it back before you next touch that area. Use BEFORE editing to pull team context (`context get` / `context search`; drifted topics carry a freshness badge), and AFTER discovering a gotcha or decision to persist it (`context update` / `context add`). On a PR the Auditor reviews the change against recorded context and comments only on a finding; it informs, never blocks. Triggers in repos with `.driftless/` or an `AGENTS.md` mentioning Driftless, and on phrases like "starting work", "how does X work in this repo", "found a gotcha", "about to push", "context get", "context add", "driftless".
license: MIT
metadata:
  author: Driftless
  version: 3.12.0
  homepage: https://driftless.icu
  cli: "@driftless-sh/cli"
---

# Driftless

Cloud is the source of truth. You pull team context before coding and persist what you learn back to Cloud. You do not own topics — the team does.

Driftless is a team-memory layer. It informs; it never blocks. Topics are anchored to globs. The core loop is simple: after a session you persist what you learned; before touching an area you pull what the team already knows. Drift surfaces as a freshness badge on the topic itself when you retrieve it — not as a report you must go fetch. On a PR, the Auditor comments only when it finds something — and the recorded context for the affected topics rides along with the finding. No finding, no comment; it never scores coverage.

## When to use this skill

- Starting or resuming a session in any repo that contains `.driftless/` or has an `AGENTS.md` mentioning Driftless.
- About to edit any file or module — you need the team's prior context (what / how / gotchas / decisions / invariants) before touching code.
- You returned after time away — when you pull a topic, a freshness badge tells you if its code changed while you were gone.
- You discovered a gotcha, a decision, an invariant — something durable that future agents and teammates should know.
- About to commit or push — you want to refresh context for your local diff and update anything that drifted.
- You suspect the context layer itself may be untrustworthy (stale, orphaned, zombie, draft-heavy).

## CRITICAL: First move per situation

Before anything else, route on the situation:

| Situation | First command |
|---|---|
| Empty workspace / your first topics | UC0 — `driftless login` then `driftless install-skill`, then just work and persist what you learn (UC2) |
| About to edit a specific file (you have topics) | `driftless context get --files "<path>"` — drifted topics show a ⚠ badge inline (UC1) |
| Looking for context on an area | `driftless context search <kw>` / `context get <slug>` (UC1) |
| Learned something worth keeping | UC2 — `driftless context update <slug>` (or `add` if no topic) |
| About to commit or push | `driftless context get --diff` (UC3); add `--mark` to flag matched topics drifted |
| Need to know if context itself is trustworthy | `driftless context doctor` |

The everyday loop is two moves: **persist what you learn after a session** (UC2), and **pull it back before you touch that area again** (UC1). You do not start with a command — early on there is little to pull, so you mostly persist; retrieval starts paying off once you have a handful of topics.

`driftless sync` is an **optional** team-wide digest — drifted topics, team PR activity, suggested topics — for when you want to scan everything that moved at once. You almost never need it, and you do not run it to start: drift already reaches you as a freshness badge when you `context get` the area you are working on (UC1).

Drift is **scoped to tracked branches**: a topic only goes stale when its covered code changes on the repo's default branch (auto-detected — main/master/whatever) or an extra branch the team opted into. A push to a throwaway feature branch is recorded but never creates false drift. `driftless branches` shows/sets the tracked set; `sync` prints it as the `tracking:` line.

`driftless context doctor` audits the context layer itself — it flags `stale`, `orphaned` (repo deleted), `zombie` (anchor patterns no longer match any file in this checkout), `draft` (suggested, never confirmed), `docs_pending`, and `repo_leak` topics. Run it if `context get` results look wrong or before relying heavily on the context layer.

---

## Use Case 1 — Retrieve context BEFORE editing code

**Trigger:** You are about to modify any module, feature, endpoint, service, or package boundary.

**Steps:**

1. **Pull context directly — no sync ritual.** Drifted topics carry a ⚠ freshness badge in every result, so you don't run `sync` first. Go straight to retrieval:

2. **If you know the slug**, get the topic directly. Otherwise search:
   ```bash
   driftless context get <slug>
   driftless context search <keyword>
   driftless context search <keyword> --tag <name>   # scope to a category (e.g. --tag billing)
   driftless context list --tag <name>               # list a whole category
   ```
   `search` returns the most relevant matches first (ranked, capped at 50) — narrow with more terms (`"quoted phrases"`, `OR`, `-negation`) rather than paging. `--tag` is repeatable (OR semantics) and works on both `search` and `list` — when the user names a category ("search in billing"), scope with `--tag` instead of hoping the keyword hits. `driftless tags` lists the workspace's tag registry with topic counts; `driftless tags add <name> --description "..."` pre-creates one (tags also auto-register the first time a topic uses them — never an error). `context list` shows the 40 most relevant (drifted first) and prints `Showing N of M`; pass `--all` or `--limit N` for more (`--json` is uncapped). This ranking + cap is enforced **server-side** (at the data surface), so the MCP `context_list` tool and the dashboard get the same bounded, drifted-first list — with `has_more` when truncated. List items are lightweight summaries; pull a topic's full body with `context get`.

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

### Auto-pull (experimental, Claude Code) — UC1 without remembering

`driftless install-skill` installs a Claude Code **PreToolUse hook** that does UC1 for you: before every `Edit`/`Write`, it injects the team's **Knowledge** for that file automatically. It is **inert until a workspace admin turns it on** at the dashboard → Settings → Automations.

- Only **Knowledge** (`reviewed`) auto-injects (a Note is a hint, not truth — it never auto-injects). This is the payoff of governance: adding a note to knowledge is what makes it reach the agent.
- Deduped per session (the same area injects once, not on every edit), size-capped (a signal-first brief under a fixed token budget), and a silent no-op when nothing matches.
- Local-first: context is served from a per-machine topic snapshot synced in the background (delta-sync against Cloud), so an edit never waits on the network — milliseconds, not round-trips. Cloud stays the source of truth.
- It complements — never replaces — UC1. If you need context for an area you're reasoning about (not editing), still pull it explicitly with `context get`.
- Controlled by the workspace toggle (`settings.auto_pull_context`); `driftless hooks disable` removes the hook from a machine entirely.

**Write-after (the symmetric half).** The same install also wires two write-side hooks, gated by the same toggle: a **PostToolUse** hook observes which files you edit, and a **Stop** hook fires once at session end — if you worked in a source area with **no recorded context** (a real gap), it nudges you to persist what you learned, with a suggested `--pattern` derived from the files you actually touched. Read-before / write-after, one mechanism: the agent stops relying on remembering to persist. Claude-Code-only; other agents persist via the CLI as usual (UC2).

---

## Use Case 2 — Persist what you learned

**Trigger:** You learned something durable — a gotcha, a decision, a constraint, an invariant — that future agents or teammates should know.

**Rule:** If it would have saved you time to know this upfront, save it now. Do NOT keep discoveries in conversation memory only. Cloud is the source of truth.

**The durability litmus — what deserves a topic:** *would a future code change meaningfully contradict this?* If yes it's durable — the *why* the code can't show (a decision, an invariant, a gotcha that outlives the line that prompted it). If it merely restates the code or is a transient value, it's NOT a topic. The structured fields (`decisions` / `gotchas` / `invariants`) hold that durable *why* — they are **NOT a changelog**: dates, "done / shipped / TODO", commit/PR refs, and roadmap items belong in git, not the topic. A field that reads like a changelog is the #1 cause of walls.

**Persisting is two moves.** First distill the durable insight (what would you tell the next agent before it touches this area?). Then choose the operation by *intent* — this is the whole game:

- **Adding a genuinely new, standalone fact** → **append** it (`--gotcha` / `--decision` / `--invariant` / `--check`). Atomic and concurrency-safe; don't regenerate a whole topic to add one line.
- **Correcting, refining, or the field already overlaps what's there** → **rewrite** it: `context get` → fold the new understanding into the old, dropping what's dead → `update --content` (or `--decisions` / `--gotchas` to replace one field). You leave the field coherent, never a pile of stale + new.

Never clobber a Knowledge (`reviewed`) topic either way — open a Suggested edit (`pr`) that carries the rewrite (see Governance below). Append grows a topic; rewrite keeps it true. Together they make a wall impossible: the moment a field overlaps or goes stale, you rewrite instead of stacking another append.

**Write the topic as markdown `--content`.** The content body IS the topic: a clear, readable explanation, like a good doc. `--tags` are **optional** — see *Tagging discipline* below; most topics need zero or one.

Keep `--what` to one tight sentence (the summary shown in lists) and put the real explanation in `--content`. `--how` is a SHORT (1–3 sentence) note on the approach or mechanism — NOT a place to paste a document; the long-form doc goes in `--content`, where it renders as markdown (headings, tables, code). A doc dumped into `--how` renders as a wall of literal markdown, and `context doctor` flags it as `mis-shaped how`.

The structured fields (`--gotcha`, `--decision`, `--invariant`, `--check`) are **optional highlights** — add one only when you want that specific thing surfaced to the machine (the PR bot, a future agent's brief) *on top of* the content. **Never invent an empty gotcha/decision just because the flag exists.** Anchors (`--pattern`) are a separate axis — they're the file-matching, and they're always worth setting.

**Tagging discipline — tags are an OPTIONAL, transversal, controlled vocabulary.** A tag is not a topic's home and not a paraphrase of its subject — it's a *cross-cut* that groups topics that share a property *across* different subjects (`security`, `performance`, `decision`, `tech-debt`). Before you coin one, run `driftless tags` and **reuse** an existing tag; create a new one only when a genuine cross-cut is missing from the registry. **Most topics need zero or one tag.** Do NOT mint a tag for the topic's subject (`billing`, `auth`, `mcp`, `dashboard`…) — that just restates what the topic is already about, and minting one per topic is exactly how a registry rots into hundreds of single-use labels nobody can navigate. The CLI auto-registers an unknown tag and never errors, so nothing stops you at write time — the discipline is yours to hold.

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
  --rel depends_on:stripe-webhook-ingest
```

Append flags (`--gotcha`, `--decision`, `--invariant`, `--check`) are repeatable and additive — every value lands atomically under a row lock, so concurrent agents don't lose appends.

**Pattern validation at write time.** Every `--pattern` and `--add-pattern` is globbed against your local checkout before the API call. The CLI prints, per pattern:

- `✓ pattern "<glob>" matches N files — e.g. <samples>` when the glob lands on real source
- `⚠ over-broad anchor` when one pattern matches more than 100 files
- `⚠ pattern "<glob>" matches 0 files — likely unintended` when every match falls under `node_modules/`, `dist/`, `.git/` etc
- **Blocking error** when a pattern matches 0 real files. The create/update is aborted; fix the glob or use `--where` for an explicit path

Trust the validator. If it says 0 matches, the topic would have been instantly stale on day one.

### CRITICAL: Correcting or consolidating a topic (the rewrite path)

Append grows a topic; **rewrite keeps it true.** Rewrite the moment you spot a **wall** — any of:

- the same point restated across several bullets (redundancy);
- an old claim and its correction both present (contradiction);
- changelog-style entries — dates, "shipped", commit refs (those belong in git, not the topic).

Recipe: `context get <slug>` (pull fresh — a rewrite is read-modify-write and can lose a concurrent append, so never rewrite from stale memory) → integrate the new understanding into the old, dropping what's dead → `update --content "..."` (or `--decisions` / `--gotchas` to replace a single field). For a Knowledge (`reviewed`) topic the same rewrite goes through a Suggested edit (`context pr <slug> --open --content @new.md`) — see Governance.

```bash
driftless context get billing-flow                     # read the current field first
driftless context update billing-flow \
  --decisions "Async webhook handler — absorbs Stripe retry storms; supersedes the earlier sync note"
# one coherent decision replacing three overlapping ones — not a fourth append
```

The append/replace mechanics of every flag are in `references/topic-anatomy.md`.

**Secret hygiene — never persist a credential.** Topics are team-visible and not encrypted at rest, so a write that carries something secret-shaped (a provider API key, a private-key block, a JWT, a live Driftless key) is **blocked**. The CLI scans locally before sending; the API enforces the same scan as the authoritative gate for every client (MCP/agent, raw API). On a genuine false positive, override with `--allow-secrets` (CLI) / `allow_secrets: true` (MCP). The block masks the match and never echoes the secret (`SECRET_DETECTED`).

### If no topic covers this area

Write the `--content` body yourself. There are three starter templates in `.driftless/assets/templates/` (`reference`, `decision`, `runbook`) — use one as scaffolding if it helps, but the content is what matters, not the form. Add a `--tags` cross-cut only if one fits (*Tagging discipline* above) — most new topics need none. Every topic lives in the workspace and is visible to the whole team (need isolation? use a separate workspace).

```bash
# Engineering knowledge — code, systems, integrations, domain maps
driftless context add billing-flow \
  --content @.driftless/assets/templates/reference.md \
  --pattern "src/billing/**" --pattern "src/checkout/**"

# Architecture/product decision (ADR) — context/options/decision/consequences
driftless context add async-webhook-handler \
  --content @.driftless/assets/templates/decision.md

# Runbook — trigger/symptoms/diagnosis/resolution/rollback
driftless context add stripe-webhook-replay \
  --content @.driftless/assets/templates/runbook.md
```

The `--content` body is the topic. A `--tags` cross-cut is optional and should reuse the registry (*Tagging discipline* above) — e.g. `--tags decision` only if the topic shares that cut with others.

### Cross-repo

A topic can span multiple repos (`where_repos`). When the same concept lives in another repo you are working in, register it explicitly:

```bash
driftless context link <slug>
```

`context link` registers the repo you are currently in into the topic's `where_repos` and **changes nothing else**. After linking, `context get <slug>` shows `used in (N repos): …` and groups files by repo.

### Cross-workspace

A workspace is the top-level container; a topic lives in exactly one. You can belong to several workspaces and switch between them without re-authenticating (one `driftless login` registers them all under one scoped key):

```bash
driftless workspace list             # the workspaces you belong to
driftless workspace switch <slug>    # operate on another one (no re-login)
driftless context move <slug> --to <workspace-slug>   # re-home a topic
```

`context move` resets the topic to a Note in its new workspace (Knowledge status doesn't carry across workspaces) and needs owner/admin in the source. Unlike `context link` (same topic, many repos, one workspace), `move` changes which workspace owns the topic.

### CRITICAL: Naming discipline

A topic has three text fields that people constantly mix up — get them right and the dashboard reads like a system map instead of slug soup:

- **`slug`** (the topic name you pass to `add`) — the stable kebab-case **id**. Specific, lowercase. e.g. `mcp-architecture`.
- **`--title`** — the **human display name**: a short noun, like a label on an architecture diagram. **One or two words, not a sentence.** e.g. `MCP`, `Billing`, `Auth`. This is what a human scans in a list.
- **`--what`** (+ `--content`) — the **explanation**. One tight sentence in `--what`; the full doc in `--content`. The explanation goes HERE, never in the title.

```bash
# GOOD — readable map
driftless context add mcp-architecture --title "MCP" \
  --what "Remote-first MCP architecture exposing topics to ChatGPT and Claude." --content @doc.md

# BAD — the title is a sentence / the slug, the what is empty
driftless context add "how the mcp works and connects to chatgpt"
```

If you omit `--title`, it defaults to the slug — which reads badly. **Always set a short `--title`.**

### CRITICAL: Anchoring discipline

**An anchor is a contract, not a convenience.** It declares which boundary a topic *governs* — and that powers everything downstream: drift fires only when the anchored boundary changes, and the PR bot + `context get --files` deliver the topic only to work *inside* it. A loose anchor poisons both — false drift on unrelated changes, and the topic leaking into work it has nothing to say about.

**Anchor to ONE boundary — a single service or module.** That boundary is a subtree in a monorepo (`apps/api/billing/**`) or a whole repo — same idea, two granularities. A concept that genuinely spans two services takes **two anchors**, not one wide glob. Size is the tell:

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

**The hub-and-spoke shape.** A whole domain is not a topic — it's a *hub* with *spokes*. Tempted to write `api-backend --pattern "apps/api/**"`? That one topic drifts on every change anywhere in the backend and leaks into work it has nothing to say about. Split it:

```text
❌  api-backend  --pattern "apps/api/**"        # owns everything → drifts on everything

✅  api-backend          (hub: how the services fit together — light or relations-only anchor)
     ├─ billing-service     --pattern "apps/api/billing/**"
     ├─ logistics-service   --pattern "apps/api/logistics/**"
     └─ auth-service        --pattern "apps/api/auth/**"
```

Connect the hub to its spokes with typed relations (or `[[slug]]` mentions):

```bash
driftless context update api-backend \
  --rel documents:billing-service --rel documents:logistics-service --rel documents:auth-service
```

Each spoke drifts only when *its* service changes; the hub is the map. This uses features you already have — narrow anchors + relations — just composed into one shape.

Manage patterns atomically — `--add-pattern` / `--remove-pattern` are repeatable AND idempotent:

```bash
driftless context update billing-flow \
  --add-pattern "src/billing/refunds/**" \
  --remove-pattern "src/billing/legacy/**"
```

See `references/topic-anatomy.md` for the full reference on fields, append-vs-replace semantics, and status lifecycle.

### Linking topics — mentions and typed relations

Topics aren't islands. Two complementary ways to connect them:

- **`[[slug]]` mention** — written inline in any free-text field (`--what`, `--decisions`, `--gotcha`, …). A weak, untyped backlink; dead links allowed (they rot silently). Use when a gotcha/decision only makes sense given another topic.
- **Typed relation** — `--rel <type>:<slug>`. A strong declared edge (both ends must exist). Types: `relates_to`, `depends_on`, `supersedes`, `blocks`, `implements`, `documents`, `risk_for`. Use when the relationship itself is the fact (A depends_on B).

```bash
driftless context update refund-flow \
  --gotcha "Refund webhooks arrive out-of-order — same race as [[stripe-webhook-ingest]]" \
  --rel depends_on:payment-gateway
driftless context relations <slug>     # incoming + outgoing
driftless context graph <slug>         # local relation graph
```

Both render in the graph (typed = solid edges, `[[mentions]]` = dashed); promote a mention to `--rel` when it captures a real dependency. Full comparison table in `references/topic-anatomy.md`.

### Governance — everything is one topic that evolves

There is a single primitive — a **topic** — that matures along one trust axis: **Note → Knowledge**. A **Note** is a draft (private by default; share it to the workspace and it's **Up for review**). **Knowledge** is the team's source of truth — *a note becomes knowledge once it's merged in*, and only a human can merge (agents write notes; an owner/admin merges). To change existing Knowledge, open a **Suggested edit** — a human merges it.

- **Note** — a draft / private observation (only its creator sees it; excluded from search by default; untouched notes auto-archive ~14d, recoverable). Where you think before sharing. Share it to the workspace and it's **Up for review**.
- **Knowledge** — the team's source of truth — the notes merged in. A human added it to knowledge, so it joins the team's living, code-anchored **knowledge graph** of the codebase. This is the durable truth you consume (`governance.authoritative: true`).

**Treat Knowledge (`reviewed`) as truth; a Note is a hint.** The model is *agents write notes, humans add them to knowledge* — an ownerless agent key can put a note up for review but never merge it in.

**Creation is always a Note (draft)** — for everyone, agent or human, CLI or MCP. Anyone may instead put a note **Up for review** (`--status proposed` on create, or `context propose` later) — that's a *request* to add it to knowledge, which is exactly what agents do. But `reviewed` is **never** set at create, by anyone: a note becomes Knowledge only through `context approve` (a human merges it in). No one merges their own work into the team's source of truth.

```bash
driftless context propose <slug>     # Note → Up for review (request to add to knowledge)
driftless context approve <slug>     # add to knowledge (merge it in — needs a human identity)
driftless context reject <slug> --reason "..."   # → back to a Note, with feedback to the author
driftless context archive <slug>     # retire a topic
```

**Don't overwrite Knowledge blindly.** To change the team's source of truth, open a **Suggested edit** — a proposed content change a human reviews and merges in:

```bash
driftless context pr <slug>                                   # list open Suggested edits
driftless context pr <slug> --open --summary "why" --content @new.md   # open a Suggested edit
driftless context pr <slug> --merge <id>                      # merge it in (human)
driftless context pr <slug> --reject <id>                     # close without merging
```

As an agent: when you discover something, persist it (UC2) — that lands as a Note. If a topic is already Knowledge (`reviewed`) and your finding changes it, open a Suggested edit instead of clobbering it. `approve`/`merge` require a human identity (an ownerless agent key writes notes but never merges them in).

---

## Use Case 0 — First-time setup

**Trigger:** `driftless context list` returns empty; you just ran `driftless install-skill` for the first time; or the user says "Set up Driftless", "install driftless", "driftless login".

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

3. **Create your first topic — about something REAL.** Pick the one thing about this repo you would tell a new teammate before they edit: the gotcha that bit you, the decision behind the architecture, the invariant nothing may break. Anchor it to the module it governs (the CLI validates the glob locally before writing). One batched command, not three updates:
   ```bash
   driftless context add billing-webhooks --title "Webhooks" \
     --what "Stripe webhook ingestion and its idempotency rules." \
     --gotcha "Stripe delivers at-least-once — handlers must short-circuit on a seen event.id" \
     --pattern "src/billing/webhooks/**"
   ```
   Know nothing concrete yet? Then skip the topic — work first and persist what you learn (UC2). A generic placeholder topic ("shared context map for this repo") documents nothing and rots on day one.

### Verify setup

```bash
driftless doctor
driftless context doctor
```

`driftless doctor` checks auth, connectivity, workspace, repo link, and AGENTS.md presence. `driftless context doctor` audits the topic layer (stale / orphaned / zombie / draft / docs_pending / repo_leak). If `login` or any command fails with `Unauthorized`/`403` or a connection error, see `references/troubleshooting.md`.

### How topics grow over time

You don't document everything upfront. Topics grow from one habit: **after a session, you persist what you learned** (UC2) — the gotcha that bit you, the decision you made, the invariant you had to respect.

The value compounds with the count. With one or two topics there is little to pull; from a handful onward, pulling context before you edit starts paying for itself every session — and a teammate's agent gets the same memory the next time they touch that area.

---

## Use Case 3 — Pre-commit refresh

**Trigger:** About to commit, push, or wrap up the task.

```bash
driftless context get --diff            # topics matching your local uncommitted changes — drifted ones show a ↻ badge
driftless context get --diff --mark     # …and flag the matched topics as drifted on Cloud (local-git drift, no GitHub App)
```

If `--diff` surfaces a stale topic that your change touches, update it (UC2) BEFORE pushing. The point of pre-commit is to leave the context layer healthier than you found it.

`--mark` is the **second drift source**: it persists drift for the matched topics from your *local* git, so a team without the GitHub App still gets a re-review signal (they show up in the dashboard's Review Queue). It is **opt-in and explicit** — plain `--diff` only displays, so work-in-progress never flips team state unless you ask.

---

## Examples

### Example 1 — Starting work on a refund flow
**User says:** "I'm going to build the refund flow on top of the existing billing module."

**Actions:**
```bash
driftless context search refund                      # → no match
driftless context get billing                        # → existing topic, read decisions + gotchas
driftless context get --files "src/billing/refunds/refund.service.ts,src/billing/refunds/refund.controller.ts"
# implement…
driftless context add refund-flow \
  --content @.driftless/assets/templates/reference.md \
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

### Example 3 — A teammate is reviewing your PR
**Situation:** Your PR touches files anchored to `billing-flow`. The Auditor reviews the change against what the team recorded and, **if it has a finding**, posts ONE comment: the finding plus the recorded gotchas, decisions and invariants for the affected topics.

**What happens:** The reviewer — who isn't running the CLI — sees the signal AND the context together, so they review *with* the team's memory instead of without it. Stale topics are flagged inside the comment. A clean PR gets **no comment at all** — context arrives anchored to a real signal, never as anchor-match spam. If your change altered how the area works, update the topic so the note stays true:
```bash
driftless context update billing-flow \
  --gotcha "..." --decision "..."   # keep the team's memory honest
```

---

## Troubleshooting

For the full catalog, see `references/troubleshooting.md`.

**`driftless: command not found`** → `npm install -g @driftless-sh/cli`

**`Unauthorized` / `403`** → `driftless login --key <api-key>` (get key at app.driftless.icu → Settings → API Keys)

**`context get` returns stale content** → topic is marked stale (files it tracks changed). Read `stale_reason`. After reviewing the current code, update the topic (UC2).

**`context get --files "..."` returns no topics for a file you expect to be covered** → no topic anchors that path. Run `context list` to see what exists, then either add a pattern via `context update <slug> --add-pattern "<glob>"` or create a new topic (UC2). Alternatively, run `driftless context doctor` and look for `zombie` — the anchor may have rotted.

**`context add` errors with "pattern matches 0 files"** → the glob is wrong or the path doesn't exist in this checkout. Verify against the repo root, broaden the glob (e.g. `src/billing/**` instead of `src/billing/*.ts`), or use `--where` for an explicit path.

**`sync` prints nothing under "stale"** → either no drift since you last looked (healthy), or the GitHub App is not installed (Cloud has no push events to compute drift from). Check `driftless doctor` — the GitHub App line is `INFO` when not installed; install at app.driftless.icu → Settings → Integrations.

**PR comment never appears on new PRs** → by design the only PR comment is the Auditor's, and it posts ONLY when it has a finding — a clean PR is silent. If you expected a finding: check the GitHub App is installed and the repo linked (`driftless doctor`), and that the Auditor is configured (dashboard → Settings → Agents — it needs a model key).

---

## Performance Notes

- **Quality over speed.** Narrow anchors, batched updates (one PATCH, not N), real causal `[[links]]` — a noisy topic rots, a precise one compounds.
- **Persist after every session** (UC2) and **trust the pattern validator** — a gotcha left in conversation memory is lost; a 0-match glob is a zombie on day one.
- **Append to add, rewrite to consolidate; never a catch-all `src/**`** — a domain is a hub with per-service spokes, not one wide glob.

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
- `--status proposed` — put a new topic **Up for review** (omit → born a Note/draft). `reviewed` is not settable at create; a note becomes Knowledge only via `context approve` (a human merges it in).
