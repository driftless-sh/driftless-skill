# Driftless CLI — Command Reference

## Installation

```bash
npm install -g @driftless-sh/cli
```

## Authentication

Driftless CLI uses API keys. No browser OAuth. No session management.

**Priority order:**
1. `DRIFTLESS_API_KEY` environment variable
2. `~/.driftless/config.json` file

```bash
# Save key to config file
driftless login --key drift_xxx

# Or via environment variable (recommended for CI and agents)
export DRIFTLESS_API_KEY=drift_xxx
```

Get your API key at app.driftless.icu → Settings → API Keys.

---

## Commands

### `driftless doctor`

Checks that the environment is fully configured before doing anything else.

```bash
driftless doctor
driftless doctor --json   # machine-readable: { ok, summary, checks[], workspace_diagnostics? }
```

Verifies: CLI auth, API connectivity, git workspace, repo connection, AGENTS.md, GitHub App, and topic anchor health. The Workspace line shows the resolved slug and its source (`config` cache, `/me`, or `git-org`). On failure it prints structured diagnostics (tried slugs, `/me` status, failed endpoint).

Run this first when starting in a new environment or debugging a broken setup. `--json` for agents/CI that parse the result.

---

### `driftless context`

Query and manage context topics — the team's shared repo knowledge.

#### List all topics

```bash
driftless context list                 # top 40, drifted-first (default cap to protect your context)
driftless context list --all           # every topic, no cap
driftless context list --limit 100     # cap to N rows (note: --json is uncapped by default)
driftless context list --stale         # only topics whose anchored code changed
driftless context list --suggested     # only not-yet-authoritative topics (draft + proposed)
driftless context list --docs          # only topics with an anchored doc
driftless context list --manual        # only manually-created topics
driftless context list --auto          # only auto-created topics (scoped to the current repo)
driftless context list --status draft  # filter by exact status
driftless context list --repo <id>     # only topics scoped to a given repo
```

`list` defaults to the 40 most relevant (drifted first, then most-recently updated) and prints `Showing N of M` when it caps — use `--all` or `--limit` to see more. `--json` returns the full set by default.

`context search` returns the most relevant matches (ranked, capped at 50). Narrow with more terms rather than paging — `"quoted phrases"`, `OR`, and `-negation` are supported.

#### Get specific context

```bash
driftless context get <slug>
```

Returns: `what`, `how`, `where`, `used_by`, `gotchas`, `decisions`, `content`, `ownership`, `invariants`, `required_checks`, `history` (recent events).

Example:
```bash
driftless context get billing
```

```
Name:      billing
What:      Handles all subscription and payment processing
How:       BillingModule → BillingService → StripeAdapter. Never call Stripe directly.
Where:     src/billing/
Gotchas:   Stripe webhook events must be idempotent — retries happen. Always check event.id before processing.
Decisions: We use Stripe's customer portal instead of building our own to reduce PCI scope.
Updated:   2026-05-10 (FILE_CHANGED: src/billing/billing.service.ts)
```

#### Search by topic

```bash
driftless context search <query>
```

Use when you don't know the exact slug.

```bash
driftless context search "payment"
→ billing, payment-gateway, stripe-adapter
```

#### Create a topic

```bash
driftless context add "<slug>" \
  --content "# Module Name\n\n## What\nWhat this module does\n\n## How\nHow it is implemented" \
  --pattern "src/auth/**" \
  --pattern "src/users/**" \
  --tags auth,security \
  --rel depends_on:token-refresh
```

`--pattern` is **repeatable** — pass it multiple times for a multi-anchor topic. The CLI blocks creation if any pattern matches 0 files locally; emits a non-blocking `⚠ over-broad anchor` warning when the topic covers more than 100 files (healthy is 5–40 — see "Anchoring discipline" below).

Other `context add` flags: `--what`, `--how`, `--where` (single file path), `--decisions`, `--gotchas`, `--ownership`, `--status proposed` (submit as a Proposal; omit → born a Note/draft — `reviewed` is not settable at create, only via `context approve`), `--file` (content from file path), `--dry-run`.

For backward compat `--where path/to/file` still works as a single explicit anchor. Use `--pattern` for globs (the common case).

#### Update an existing topic

```bash
# Replace scalars
driftless context update <slug> --what "..." --how "..."

# Append, never clobber — repeatable
driftless context update <slug> \
  --gotcha "Discovered a new trap" \
  --gotcha "And another one" \
  --decision "Chose A over B because Y" \
  --invariant "All writes must use the lock" \
  --check "Run integration suite before merging"

# Manage patterns atomically — repeatable
driftless context update <slug> \
  --add-pattern "src/billing/webhooks/**" \
  --add-pattern "src/billing/refunds/**" \
  --remove-pattern "src/billing/legacy/**"
```

- **Singular** flag (`--gotchas`, `--decisions`, `--pattern`) = REPLACE the whole field.
- **Append** flags (`--gotcha`, `--decision`, `--invariant`, `--check`) = APPEND a new line. Pass each as many times as you want — every value is appended atomically in a single PATCH (the SQL UPDATE runs under a row lock so concurrent appends don't lose data).
- `--add-pattern` / `--remove-pattern` are idempotent: running the same one twice is a no-op. Both can fire in the same PATCH — remove applies first, then add.
- Every PATCH bumps `version` and writes a history event, so **batch related changes into ONE invocation** rather than splitting into N small ones.

Other `context update` flags: `--content` (full markdown body), `--tags`, `--status <reviewed|draft>`, `--where` (single file path), `--gotchas` (replace all gotchas), `--decisions` (replace all decisions), `--ownership`, `--pattern` (replace all patterns), `--enforce "rule"`, `--dry-run`, `--json`, `--check` (required check, repeatable).

Get `--help` on any subcommand for the full flag list:

```bash
driftless context add --help
driftless context update --help
```

#### Anchoring discipline

Patterns define which slice of the codebase a topic claims. Size matters:

| File count | Verdict |
|---|---|
| 5–40 | Healthy — narrow enough to mean something specific |
| 41–99 | Wide — verify it's truly one concept |
| 100+ | Trap — almost always over-broad; the topic becomes a useless catch-all |

The CLI emits `⚠ over-broad anchor` after `add`/`update` when the topic crosses 100 files. Don't suppress it — split into narrower topics instead.

**Good** (narrow, multi-anchor):
```bash
driftless context add "checkout-flow" \
  --pattern "src/checkout/**" --pattern "src/cart/**" \
  --what "Cart → checkout → payment"
```

**Bad** (catch-all that will rot):
```bash
driftless context add "backend" --pattern "src/**"       # don't
```

#### Linking topics with `[[slug]]`

Inside any free-text field on a topic (`--what`, `--how`, `--decisions`, `--gotcha`, `--invariant`, `--ownership`), write `[[other-topic-slug]]` to mark a forward reference. The API parses every mention on every create/update and stores the slugs in `references_topics`. The reverse direction (`referenced_by`) is computed at read-time.

```bash
driftless context update refund-flow \
  --gotcha "Same race as [[stripe-webhook-ingest]]" \
  --decision "Idempotency-key derived from charge_id, mirroring [[payment-gateway]]"
```

`context get refund-flow` then shows the forward refs; `context get payment-gateway` shows `refund-flow` under `referenced by`.

- Slug grammar: lowercase alphanumerics + hyphens, must start with `[a-z0-9]`. Anything else is ignored.
- Self-references are silently stripped.
- Dead links (slugs that don't resolve to a real topic) are accepted in writes but never produce a backlink on the other side — they sit silently in `references_topics` until you fix them.

#### Cross-repo: link a topic to another repo

```bash
driftless context link <slug>
```

Registers the repo you are currently in into the topic's `where_repos`. Touches only the repo list — `what`/`how`/`gotchas`/`decisions` are NOT modified. Use this when the same concept lives in another repo.

`context get <slug>` then shows `used in (N repos): …`. `context update` also auto-links the repo you run it from as a side effect — use `context link` when linking is all you want.

#### Sync a file or note to a topic

```bash
# Anchor a doc file — sets file_content, status=draft
driftless context sync <slug> --doc path/to/file.md

# Add a quick note — maps to the decisions field
driftless context sync <slug> --note "Key finding from recent review"

# Sync with path anchoring
driftless context sync <slug> --doc path/to/file.md --files "src/auth/**"
```

Stores the file's content as `file_content` on the topic. Useful for linking AGENTS.md sections, architecture docs, or runbooks.

#### Get context for specific files

```bash
# Takes comma-separated file paths (not globs)
driftless context get --files "src/auth/guard.ts,src/auth/service.ts"
```

Delivers context for all topics that match the given file paths. Use before starting work on a specific area. For local uncommitted changes, use `context get --diff` instead.

#### Delete a topic

```bash
driftless context delete <slug>
driftless context delete <slug> --dry-run   # preview without deleting
```

#### Move a topic to another workspace

```bash
driftless context move <slug> --to <workspace-slug>
driftless context move <slug> --to <workspace-slug> --dry-run
```

A topic lives in exactly one workspace. `move` re-homes it and **resets it to a draft** in its new workspace (a vouch doesn't carry across workspaces). Requires owner/admin in the source workspace and membership in the target. Workspace-local anchors (repo links, relations) are dropped.

#### Governance — a topic is authoritative only if approved

Lifecycle: `draft → proposed → reviewed → archived`. `reviewed` is the **authoritative** state — the read response carries `governance.authoritative: true`. Treat `reviewed` as truth, `draft`/`proposed` as a hint. The model is *agents propose, humans approve*.

```bash
driftless context propose <slug>     # submit a draft for review
driftless context approve <slug>     # make it authoritative (needs a human identity)
driftless context reject <slug>      # send a proposed topic back to draft
driftless context archive <slug>     # retire a topic
```

**topic-PR** — propose a content change to an approved topic instead of overwriting it; a human reviews and merges:

```bash
driftless context pr <slug>                                        # list open proposals
driftless context pr <slug> --open --summary "why" --content @new.md   # open a proposal
driftless context pr <slug> --merge <id>                           # apply + approve (human)
driftless context pr <slug> --reject <id>                          # close without applying (human)
```

`approve` / `merge` / `reject` require a human identity — an ownerless agent key can propose but never bless. If a `reviewed` topic needs to change, open a `pr`; don't clobber it.

#### Share a topic publicly

```bash
driftless context share <slug>            # public read-only link (driftless.icu/s/<token>)
driftless context share <slug> --revoke   # turn the link off
```

---

### `driftless workspace`

Navigate the workspaces you belong to. A workspace is the top-level container for context; a user can belong to many. `driftless login` registers all of them under one **scoped key**, so you switch without re-authenticating.

```bash
driftless workspace list            # the workspaces you belong to (● = active)
driftless workspace current         # the active workspace
driftless workspace switch <slug>   # operate on another one (no re-login)
driftless workspace add --key <drift_…>   # register a workspace by its API key
```

Scoped credentials: one API key can be authorized for several workspaces (set the **Authorized workspaces** when you create the key in the dashboard). Access is always gated by your live membership — being removed from a workspace revokes the key there immediately. Switching to a workspace your key isn't scoped to surfaces a 403 with guidance.

---

### `driftless sync`

Pull Cloud state for the current repo. Run this at the start of every session — before touching any code.

```bash
driftless sync           # Cloud pull for current repo
driftless sync --json    # machine-readable output
```

Reports — the deduped "what moved around my topics" signal, not a raw event feed:
- **Stale topics** — a topic whose covered code the team changed on a tracked branch since you last looked
- **Team PR activity** — what teammates shipped against this repo's context (via the GitHub App)
- **Tracking line** — which branches' pushes count as drift (the default branch is always tracked)

No local diff. Pure Cloud state. Use `driftless context get --diff` when you need topics for your current *local* uncommitted changes.

---

### `driftless branches`

View or set which branches' pushes count as drift. The repo's **default branch** (from GitHub, auto-detected — main/master/whatever) is always tracked: zero config. Add extra branches only if your real integration branch isn't the GitHub default (e.g. you ship from `release` or `develop`).

```bash
driftless branches                 # show tracked branches
driftless branches add release     # also detect drift from pushes to `release`
driftless branches rm staging
driftless branches --json
```

A push to a non-tracked branch is recorded for audit but does **not** mark topics stale — a throwaway feature branch never creates false drift. This is why `sync` only surfaces drift from tracked branches.

---

### `driftless context doctor`

Audits the context layer itself — answers "can I trust `context get`?".

```bash
driftless context doctor
driftless context doctor --json
```

Categories: `stale` (code changed, context not updated), `orphaned` (repo deleted — hidden from agents), `draft` (suggested, never confirmed), `docs_pending` (doc anchored, never synced), `repo_leak` (references an unknown repo id). Exits non-zero when `orphaned` or `repo_leak` are present.

---

### `driftless install-skill`

Writes the Driftless AGENTS.md template into the current repo so AI agents (Claude Code, Cursor, Codex) pick it up automatically.

```bash
driftless install-skill
```

Creates or appends to `AGENTS.md` at the repo root.

---

## Environment variables

| Variable | Default | Description |
|---|---|---|
| `DRIFTLESS_API_KEY` | — | API key for authentication |
| `DRIFTLESS_API_URL` | `https://api.driftless.icu/api/v1` | Cloud API endpoint |

---

## Exit codes

| Code | Meaning |
|---|---|
| `0` | Command succeeded |
| `1` | Command failed |
