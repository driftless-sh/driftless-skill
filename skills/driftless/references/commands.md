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
driftless context list --suggested     # only not-yet-Knowledge topics (Notes + Up for review)
driftless context list --docs          # only topics with an anchored doc
driftless context list --manual        # only manually-created topics
driftless context list --auto          # only auto-created topics (scoped to the current repo)
driftless context list --status draft  # filter by exact status
driftless context list --repo <id>     # only topics scoped to a given repo
driftless context list --tag <name>    # only topics in a category (repeatable; OR)
```

`list` defaults to the 40 most relevant (drifted first, then most-recently updated) and prints `Showing N of M` when it caps — use `--all` or `--limit` to see more. `--json` returns the full set by default.

`context search` returns the most relevant matches (ranked, capped at 50). Narrow with more terms rather than paging — `"quoted phrases"`, `OR`, and `-negation` are supported. Add `--tag <name>` (repeatable) to scope the search to a category.

#### Tags — the category registry

```bash
driftless tags                                  # list tags with topic counts (paginated, most-used first)
driftless tags add <name> --description "..."   # pre-create a tag (idempotent)
driftless tags rename <old> <new>               # rename everywhere (rewrites topics; owner/admin)
driftless tags rm <name>                        # delete + detach from every topic (owner/admin)
```

Tag names normalize on every surface (lowercase, spaces→dashes), so `Billing` and `billing ` are the same tag. Attaching a tag that was never pre-created auto-registers it — the registry is curation, never a gate.

#### Areas — the topic's domain home

```bash
driftless area list                              # list areas (id, name, topic count) — run this to find the right --area
driftless area add "<name>" [--description "..."] # create a domain
```

Every topic files into exactly one **area** (rule B) — its ontological home (e.g. `Billing`, `Platform & Infra`), distinct from transversal **tags**. File a topic with `context add/update --area <name|id>` (the name from `area list`, or its id). `area list` is the canonical way to discover area names — `context list` carries the `area_id` but not the name, so resolve it here, not by guessing. Areas are id-keyed: renaming one never re-files its topics.

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
  --tags security \
  --rel depends_on:token-refresh
```

`--pattern` is **repeatable** — pass it multiple times for a multi-anchor topic. The CLI blocks creation if any pattern matches 0 files locally; emits a non-blocking `⚠ over-broad anchor` warning when the topic covers more than 100 files (healthy is 5–40 — see "Anchoring discipline" below).

Other `context add` flags: `--what`, `--how`, `--where` (single file path), `--decisions`, `--gotchas`, `--ownership`, `--status proposed` (put it **Up for review**; omit → born a Note/draft — `reviewed` is not settable at create, a note becomes Knowledge only via `context approve`), `--file` (content from file path), `--dry-run`, `--allow-secrets` (override the secret-scan; see below).

For backward compat `--where path/to/file` still works as a single explicit anchor. Use `--pattern` for globs (the common case).

**Secret hygiene.** Topics are team-visible and not encrypted at rest, so a write is **blocked** when any field looks like a credential — a provider key (AWS/GitHub/OpenAI/Anthropic/Stripe/Slack/Google/…), a private key block, a JWT, or a live Driftless key. The CLI scans locally before sending (the secret never leaves your machine); the API enforces the same scan as the authoritative gate for every client (MCP/agent, raw API). On a false positive, override with `--allow-secrets` (CLI) / `allow_secrets: true` (MCP / API body). The block masks the match — it never echoes the secret. Error code: `SECRET_DETECTED`.

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

- **Plural/replace** flags (`--gotchas`, `--decisions`, `--invariants`, `--checks`, `--pattern`) = REPLACE the whole field.
- **Append** flags (`--gotcha`, `--decision`, `--invariant`, `--check`) = APPEND a new line. Pass each as many times as you want — every value is appended atomically in a single PATCH (the SQL UPDATE runs under a row lock so concurrent appends don't lose data).
- `--add-pattern` / `--remove-pattern` are idempotent: running the same one twice is a no-op. Both can fire in the same PATCH — remove applies first, then add.
- Every PATCH bumps `version` and writes a history event, so **batch related changes into ONE invocation** rather than splitting into N small ones.

Other `context update` flags: `--content` (full markdown body), `--tags`, `--status <reviewed|draft>`, `--where` (single file path), `--gotchas` (replace all gotchas), `--decisions` (replace all decisions), `--invariants` (replace all invariants, repeatable = whole array), `--checks` (replace all required_checks, repeatable = whole array), `--ownership`, `--pattern` (replace all patterns), `--enforce "rule"`, `--dry-run`, `--json`, `--check` (required check, repeatable), `--allow-secrets` (override the secret-scan — see "Secret hygiene" above).

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

**Mentions vs typed relations.** `[[slug]]` is the *weak* link (a passing reference). A *typed* relation (`context update <slug> --rel depends_on:other`) is the strong, declared edge. **Both now render in `context graph`, `context relations`, and the dashboard graph** — typed as solid edges, mentions as faint dashed `mention` edges. Use `[[…]]` when a note only makes sense given another topic; use `--rel` when the relationship itself is the fact.

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

A topic lives in exactly one workspace. `move` re-homes it and **resets it to a Note** in its new workspace (Knowledge status doesn't carry across workspaces). Requires owner/admin in the source workspace and membership in the target. Workspace-local anchors (repo links, relations) are dropped.

#### Governance — a topic is authoritative only once it's been added to knowledge

A topic matures along one trust axis: **Note → Knowledge** (status enum: `draft → proposed → reviewed → archived`). A **Note** is a draft (private by default; share it to the workspace and it's **Up for review**). **Knowledge** (`reviewed`) is the team's source of truth — *a note becomes knowledge once it's merged in* — and the read response carries `governance.authoritative: true`. Treat Knowledge as truth, a Note (`draft`/`proposed`) as a hint. The model is *agents propose; merging into Knowledge is an owner/admin act* — you can run the merge, but only when an owner/admin explicitly asks.

```bash
driftless context propose <slug>     # Note → Up for review (request to add to knowledge)
driftless context approve <slug>     # add to knowledge — merge it in (owner/admin only; run only when asked)
driftless context reject <slug>      # send an up-for-review topic back to a Note
driftless context archive <slug>     # retire a topic
```

**Suggested edit** — propose a content change to a Knowledge topic instead of overwriting it; an owner/admin reviews and merges it in:

```bash
driftless context pr <slug>                                        # list open Suggested edits
driftless context pr <slug> --open --summary "why" --content @new.md   # open a Suggested edit
driftless context pr <slug> --merge <id>                           # merge it in (owner/admin)
driftless context pr <slug> --reject <id>                          # close without merging (owner/admin)
```

`approve` / `merge` / `reject` require owner/admin authority — an ownerless agent key or a non-owner writes notes but is refused the merge. Don't merge unprompted: do it only on an explicit owner/admin request. If a Knowledge (`reviewed`) topic needs to change, open a Suggested edit (`pr`); don't clobber it.

#### Share a topic publicly

```bash
driftless context share <slug>            # public read-only link (driftless.icu/s/<token>)
driftless context share <slug> --revoke   # turn the link off
```

#### Comment on a topic, a record, or a card

A comment is a thin annotation that points AT a spine object (a topic/record) or a project card — **not** a topic. It carries no governance of its own and resolves into an edit (`open → resolved → wont_fix`). Author is you (human) or your agent. The killer placement is the Note→Knowledge gate: leave fine-grained, field-level feedback at review time instead of a coarse approve/reject.

```bash
# Comment on a topic (by slug); --field scopes it to one field (e.g. decisions)
driftless context comment add billing-flow --body "This decision contradicts the refund invariant" --field decisions

# Comment on a project card or a record (by uuid)
driftless context comment add --card <card-uuid> --body "Blocked on the webhook fix"
driftless context comment add --record <record-uuid> --body "Duplicate of the Globex lead"

# List a topic's comments at review time (server-side filtered by slug)
driftless context comment list billing-flow --status open

# Resolve when an edit addresses it (or won't-fix / reopen)
driftless context comment resolve <comment-id>
driftless context comment resolve <comment-id> --wont-fix
driftless context comment reopen  <comment-id>
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

### `driftless project`

Projects are boards of cards — shared execution environments for humans and agents.

```bash
# Projects
driftless project add "<title>"
driftless project list [--status active|done|archived]
driftless project get <id>
driftless project update <id> [--title "..."] [--status active|done|archived]

# Cards — list
driftless project cards <project-id> [--status todo,in_progress,...] [--limit n]

# Cards — agent loop tick (one call per iteration)
driftless project card next <project-id>
# Returns: next actionable card + context_bundle + project_done flag

# Cards — create
driftless project card add <project-id> \
  --title "..." \
  [--description "..."] \
  [--acceptance "what done means"] \
  [--validate "pnpm test billing"] \
  [--dep <card-id>]              # repeatable — sets depends_on
  [--owner <clerk-id>]

# Cards — read / update / delete
driftless project card get <project-id> <card-id>
driftless project card status <project-id> <card-id> <status>
driftless project card move <project-id> <card-id> <to-project-id>
driftless project card rm <project-id> <card-id>
```

**Card fields:**
- `acceptance` — what "done" means (free text). Agent checks this before marking done.
- `validate` — shell command to verify the card. Non-zero exit = stop-and-fix.
- `depends_on` — list of card UUIDs. A `todo` card is "ready" only when all deps are `done`.
- `ready` — computed field (true when todo + all deps done). Never set manually.

**Card statuses:** `todo` · `in_progress` · `blocked` · `review` · `done`

**Agent loop:** call `next` → work → validate → persist → mark done → call `next`. Repeat until `project_done: true`. See "Working through a project" in SKILL.md.

---

### `driftless collection`

The object-native operational substrate. A **Collection** is a configured table (`record_schema` + `views` + lifecycle + `criterion_rel_slugs`); a **Record** is one typed row in it. Each use case — a CRM, a bug tracker, support tickets, a content calendar — is a *configured Collection*, not new code. There are three archetypes: **pipeline** (records flow by status), **analysis** (record anchored to a drifting source), **content** (a record with a draft→published lifecycle).

> Collections are a **separate primitive from `project`** (the agent execution loop). They coexist — a Collection is operational data a human runs augmented by AI; a Project is where an agent works a loop.

```bash
# Create a Collection. record_schema/views accept inline JSON or @file.
driftless collection add "Sales Pipeline" --archetype pipeline \
  --schema '[{"key":"company","type":"text"},{"key":"mrr","type":"currency"},{"key":"stage","type":"status","stages":["new","qualified","won","lost"]}]' \
  --views  '[{"type":"board","group_by":"stage"}]' \
  --criterion "how-we-sell,icp"        # Knowledge topics the work reads (the seam, coming down)

driftless collection list [--status active|archived]
driftless collection get <id>
driftless collection update <id> [--name "..."] [--status active|archived] [--schema @file] [--views @file]

# Records — typed rows. status is validated against the Collection's stages.
driftless collection record add <collection-id> --fields '{"company":"Acme","mrr":500}' --status new
driftless collection records <collection-id> [--status <stage>]
driftless collection record get    <collection-id> <record-id>
driftless collection record update <collection-id> <record-id> [--fields '<json>'|@file] [--status <stage>]
```

**Field types (`record_schema`):** `text · long_text · number · currency · date · datetime · select · multi_select · status(stages[]) · relation · user · url · email · phone · file · anchor_ref · ai_field`. A `status` field's `stages[]` defines the record lifecycle — a record's `status` must be one of them.

**View types (`views`):** `board · table · list · calendar · gallery` (with `group_by`, `filter`, `sort`, `visible_fields`).

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
