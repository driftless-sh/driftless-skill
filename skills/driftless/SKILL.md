---
name: driftless
description: Driftless is your team's shared memory in any repo — durable, area-filed, code-anchored notes you READ before working and WRITE as you learn. It is a vault: most notes stay notes, a few become team Knowledge. Pull context before editing a module (`context get` / `context search` / `--files`; drifted topics show a freshness badge); leave one clean note after you learn a gotcha, decision, or invariant (`context add` / `context update`). Quality is enforced at write time — one concept per note, filed in an area, narrowly anchored, the durable why (never a changelog). Use in repos with `.driftless/` or an AGENTS.md mentioning Driftless, and on phrases like "what do we know about X", "save this", "found a gotcha", "starting work", "about to push", "context get", "driftless".
license: MIT
metadata:
  author: Driftless
  version: 4.0.0
  homepage: https://driftless.icu
  cli: "@driftless-sh/cli"
---

# Driftless

Driftless is the team's shared memory in Cloud. You **read** it before you work and **write** clean notes into it as you learn. You don't own topics — the team does; Cloud is the source of truth.

It is a **vault**, not a filing ceremony. Most of what you write stays a Note — that's healthy and expected. A few notes become **Knowledge** (a human merges them in) — the cross-cutting truths the whole team relies on. There is no approval gate you must clear; **a note is first-class and the agent reads it.** Because there's no gate, *quality is enforced at write time* — by the six rules below. That discipline is the whole product: a clean vault is useful, a messy one rots.

## The two moves

1. **Read before you work.** Before touching a module, pull what the team already knows for that area. Drifted topics carry a freshness badge so you know what changed while you were gone.
2. **Write one clean note after you learn.** When you hit a durable gotcha, decision, or invariant, leave exactly one clean note. Don't chase approval — leave something good.

## CRITICAL: the six rules every write follows

These are not advice — they are the gate. Hold them on every `add` / `update` (the CLI and MCP nudge you, but the discipline is yours).

- **A. One concept per note.** Atomic. If you're tempted to cover two things, write two notes.
- **B. Always file into an area** (`--area`). Pick the domain before you write; if none fits, create one. Never leave a note in *Unassigned* — that's where vaults rot.
- **C. Anchor narrow** (`--pattern`, ~5–40 files, never `src/**`). For non-code memory, anchor the doc/api/system it's about. A loose anchor causes false drift and leaks the note into work it has nothing to say about.
- **D. Trust is a signal, not a gate.** Knowledge (`reviewed`) = team truth; a Note = a hint. Both reach the agent, weighted. Write to leave a *good note*, not to get something approved.
- **E. Rewrite to consolidate; append to add.** Adding a new standalone fact → append. Correcting or overlapping what's there → rewrite the field coherently. Never stack a wall.
- **F. Content is the durable *why*, not a changelog.** Decisions, invariants, gotchas a future code change would contradict. No dates, "shipped", PR/commit refs, or status — that lives in git. A changelog-shaped field is the #1 cause of slop.

The litmus for whether something deserves a note at all: **would a future code change meaningfully contradict it?** If yes, it's durable. If it just restates the code or is a transient value, it's not a topic.

## Reading — before you work

```bash
driftless context get <slug>                  # if you know it
driftless context search <keyword>            # ranked; narrow with more terms, "quotes", OR, -negation
driftless context get --files "src/billing/billing.service.ts,src/billing/refunds/*"   # match by path
driftless context list --area billing         # browse a domain
```

Read the full topic: `what` / `how` / `where` / `gotchas` / `decisions` / `invariants` / `history`. **Returning after time away?** Check `history` and the freshness badge before assuming your old understanding holds.

No topic matches the file you're about to edit? Read the code, then leave a note (don't skip it — the absence of a topic is a gap to close, not permission to assume).

> Auto-pull (Claude Code): `install-skill` wires a PreToolUse hook that injects the team's **Knowledge** for a file before you edit it — inert until a workspace admin enables it. Details in `references/workflow.md`.

## Writing — after you learn

Distill the durable insight, then `add` (no topic yet) or `update` (one exists). Batch every related change into ONE call — each write bumps version and writes a history event.

```bash
# New area — write the markdown body yourself (templates in .driftless/assets/templates/)
driftless context add refund-flow --title "Refunds" \
  --what "Refund issuance and its reconciliation rules." \
  --content @doc.md \
  --area billing \
  --pattern "src/billing/refunds/**"

# Existing topic — append new standalone facts atomically
driftless context update billing-flow \
  --gotcha "Stripe webhooks deliver out-of-order — idempotency-key required" \
  --decision "Async handler chosen over sync to absorb retry storms" \
  --invariant "Webhook handlers must be idempotent" \
  --add-pattern "src/billing/webhooks/**"

# Spot a wall (redundancy / contradiction / changelog)? Rewrite the field coherently — don't append a fix
driftless context get billing-flow                              # pull fresh first
driftless context update billing-flow --decisions "<one coherent decision replacing the overlapping ones>"
```

`--what` = one tight sentence (the list summary). `--content` = the real explanation as markdown (the body IS the topic). `--how` = a SHORT note on the mechanism (1–3 sentences, never a pasted doc). The structured flags (`--gotcha`/`--decision`/`--invariant`/`--check`) are optional highlights surfaced to the machine — never invent an empty one.

**Pattern validation runs at write time** — every glob is matched against your checkout. `⚠ over-broad anchor` (>100 files) means split into narrower topics; a 0-match glob is blocked (it'd be a zombie on day one). Trust it.

**Never persist a secret** — credential-shaped writes are blocked client- and server-side (`--allow-secrets` only for a genuine false positive).

## Naming (get these three right — they're constantly confused)

- **slug** — stable kebab-case id, specific. e.g. `mcp-architecture`.
- **`--title`** — short human display name, **one or two words** like a diagram node (`MCP`, `Billing`). Not a sentence. Always set it.
- **`--what` / `--content`** — the explanation. Never put the explanation in the title.

## Anchoring (rule C, in depth)

An anchor is a **contract**: it declares the one boundary a topic governs. Drift fires only when that boundary changes; retrieval delivers the topic only to work inside it.

| Match count | Verdict |
|---|---|
| 5–40 | Healthy |
| 41–99 | Wide — verify it's truly one concept |
| 100+ | Trap — split it |

A whole domain is **not** a topic — it's a hub with per-service spokes (`api-backend` as a light hub, `billing-service --pattern "apps/api/billing/**"` as a spoke), connected by relations. If you reach for `src/**`, the answer is more topics, not a wider glob. Full discipline in `references/topic-anatomy.md`.

## Linking

- **`[[slug]]` mention** — inline in any free-text field; a weak untyped backlink. Use when a gotcha/decision only makes sense given another topic.
- **`--rel <type>:<slug>`** — a strong typed edge (both ends must exist): `relates_to`, `depends_on`, `supersedes`, `blocks`, `implements`, `documents`, `risk_for`. Use when the relationship itself is the fact.

Both render in the graph (`context graph`). Promote a mention to `--rel` when it's a real dependency.

## Tags (optional, transversal)

A tag is a cross-cut that groups topics across *different* subjects (`security`, `performance`, `tech-debt`) — **not** the topic's subject (don't tag `billing` on a billing topic) and **not** its home (that's the area). Most notes need zero or one. Reuse the registry (`driftless tags`) before coining one.

## Governance (the footnote — most notes stay notes)

One primitive, one trust axis: **Note → Knowledge**.

- **Note** — what you write. Private by default; share it to the workspace and it's *Up for review*. Untouched notes auto-archive ~14d (recoverable). The agent reads notes as **hints**.
- **Knowledge** (`reviewed`) — the team's source of truth, *the notes merged in*. A human merges; an agent can put a note up for review but never merges its own. The agent reads Knowledge as **truth**.

```bash
driftless context propose <slug>     # Note → Up for review
driftless context approve <slug>     # merge into Knowledge (human only)
driftless context pr <slug> --open --summary "why" --content @new.md   # Suggested edit to existing Knowledge
```

You don't need to push notes toward Knowledge — that promotion is for the few cross-cutting truths the team relies on. Just leave clean notes (the six rules). To change existing Knowledge, open a Suggested edit — never clobber it.

## Setup (first time)

```bash
driftless login                # or: --key <api-key>
driftless install-skill        # writes CLAUDE.md, AGENTS.md, .driftless/
driftless doctor               # verify auth/connectivity/repo link
```

Then just work and leave clean notes. Don't seed a generic placeholder topic — it documents nothing and rots. Your first note should be about something REAL you'd tell a new teammate.

## Pre-commit / pre-push

```bash
driftless context get --diff           # topics matching your local diff; drifted ones show a badge
driftless context get --diff --mark    # …and flag them drifted from local git (no GitHub App needed)
```

If a topic your change touched is stale, refresh it before pushing. `--mark` is opt-in — plain `--diff` only displays.

## Health

`driftless context doctor` audits the vault itself (stale / orphaned / zombie / draft / docs_pending / repo_leak). Run it if retrieval looks wrong or before relying heavily on the layer.

## See also

- `references/commands.md` — full CLI reference
- `references/workflow.md` — the loop, auto-pull/write-after hooks, drift, branches
- `references/topic-anatomy.md` — fields, append-vs-replace, anchoring, linking, lifecycle
- `references/cloud.md` — Cloud data model
- `references/troubleshooting.md` — error / cause / solution catalog

Common flags: `--dry-run` (preview), `--json` (machine output), `--status proposed` (born Up for review; `reviewed` is never set at create — a human merges).
