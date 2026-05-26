# Troubleshooting

Structured catalog of common errors. Format: **Symptom** → **Cause** → **Solution**.

For topic field/kind questions, see `topic-anatomy.md`.

---

## Install & auth

### `driftless: command not found`

**Cause:** CLI not installed globally.

**Solution:**
```bash
npm install -g @driftless-sh/cli
driftless --version
```

### `Unauthorized` or HTTP 403

**Cause:** No API key set, expired key, or wrong workspace.

**Solution:**
```bash
driftless login --key <api-key>           # persist to ~/.driftless/config.json
# or set env var (preferred for CI):
export DRIFTLESS_API_KEY=drift_xxx
driftless doctor                          # verify
```

Get a fresh key at driftless.icu → Settings → API Keys.

### `driftless doctor` reports "Workspace fail"

**Cause:** The configured workspace slug no longer resolves (renamed, deleted, or the API key belongs to a different org).

**Solution:** Read the `Workspace` line — it shows the resolved slug and source. Run `--json` to see structured diagnostics:
```bash
driftless doctor --json
```
If the slug is wrong, re-authenticate with a key from the correct workspace.

---

## Topics — context get / list / search

### `No topics found`

**Cause:** No one has created topics for this workspace/repo yet.

**Solution:** Create the first topic:
```bash
driftless context add onboarding-context \
  --kind domain-map \
  --what "Shared context map for this repo." \
  --how "Captures what humans and agents need to know."
```

### `context get <slug>` returns "stale"

**Cause:** Files this topic anchors changed on a tracked branch since the topic was last updated.

**Solution:**
1. Read the `stale_reason` field (which files changed, on which branch, what commit).
2. Open the changed files and compare against the topic's `what` / `how` / `gotchas` / `decisions`.
3. Update the topic (UC2) so it matches the current code:
   ```bash
   driftless context update <slug> \
     --gotcha "..." --decision "..." --add-pattern "..."
   ```

The `stale` flag clears automatically when you next `context update` the topic.

### `context search <keyword>` returns no results

**Cause:** No topic's `name`, `what`, `how`, `tags`, or content contains the keyword.

**Solution:**
```bash
driftless context list                    # see everything
driftless context list --kind code-context
driftless context list --suggested        # also show drafts
```

If the area genuinely has no topic, create one (UC2).

### `context get --diff` returns nothing

**Cause:** Either no local uncommitted changes, or the changed files don't match any topic's anchors.

**Solution:**
- `git status` to confirm there are local changes.
- If there are changes but no match, the touched files are uncovered — UC2 candidate.
- For *team* drift (not your local), use `driftless sync` instead.

### `context get --files "a.ts" --json` returns empty `topics`

**Cause:** No topic's patterns match `a.ts`.

**Solution:**
1. `driftless context list` — find a related topic.
2. Add the file path to the closest matching topic:
   ```bash
   driftless context update <slug> --add-pattern "<glob covering a.ts>"
   ```
3. Or create a new topic for the area (UC2).

---

## Sync & drift

### `driftless sync` shows nothing under "stale"

**Cause (most common):** No drift since you last looked — healthy state.

**Cause (subtle):** GitHub App is not installed, so Cloud has no push events to compute drift from. `doctor` shows the GitHub App line as `INFO: Not installed`.

**Solution:**
- For team drift, install the GitHub App at driftless.icu → Settings → Integrations.
- For *local* uncommitted changes, use `driftless context get --diff` instead — no GitHub App required.

### `sync` shows drift on a feature branch but I never tracked it

**Cause (impossible):** Drift is scoped to **tracked branches only** (the default branch is always tracked). A push to an untracked branch is recorded for audit but does NOT mark topics stale.

**Solution:** Check what's tracked:
```bash
driftless branches
```
If you see drift unexpectedly, the branch IS in the tracked set. Use `driftless branches rm <branch>` to remove it.

---

## PR bot

### No comment appears on new PRs

**Cause:** The GitHub App isn't installed on the repo, or the repo isn't linked to the workspace.

**Solution:**
- Confirm with `driftless doctor` — the GitHub App line will read `INFO: not installed` if missing.
- Install the App at driftless.icu → Settings → Integrations.
- Confirm the repo is in the workspace: `driftless context list --auto` (or open the dashboard).

### PR comment shows everything as "outside any topic"

**Cause:** No topic anchors cover the files this PR touched.

**Solution:** That's the bot's job — close the gap. For each path in the comment's "files not anchored to any topic" section:
- If the path belongs to an existing topic, extend it:
  ```bash
  driftless context update <slug> --add-pattern "<glob>"
  ```
- If the path is a new concept, create a topic:
  ```bash
  driftless context add <slug> --kind code-context --pattern "<glob>"
  ```

The next PR touching the same area will show those files as covered.

---

## Context add / update

### `context add` rejected with "pattern matches 0 files locally"

**Cause:** The glob you passed doesn't match any file in this checkout. The validator runs `globSync` against the repo root before the API call and aborts the create/update so the topic isn't born already-stale.

**Solution:**
- Open a shell and verify the glob expands to real paths (e.g. `ls src/billing/**` with glob expansion).
- Broaden the glob if needed: `src/billing/**` instead of `src/billing/*.ts`.
- If the file isn't checked in yet, use `--where "src/billing/refunds/refund.service.ts"` to anchor by explicit path (no glob validation).

### `⚠ pattern matches only node_modules/dist/.git`

**Cause:** Every file the glob hits is under `node_modules`, `dist`, `build`, `out`, `.git`, `coverage`, `.next`, `.turbo`, or `.pnpm-store`. Almost certainly you forgot a `src/` prefix or pointed at generated output.

**Solution:** Re-anchor against source paths. The CLI treats this as a 0-real-match and blocks the write.

### `⚠ over-broad anchor: covers 187 files`

**Cause:** Your pattern matches more than 100 files — almost certainly catch-all.

**Solution:** Split into narrower topics. The warning is non-blocking, but heeding it keeps `context get` results precise.
```bash
# instead of one wide topic:
driftless context add backend --pattern "src/**"

# create multiple narrow ones:
driftless context add auth-boundary --pattern "src/auth/**"
driftless context add billing-flow --pattern "src/billing/**"
driftless context add checkout-flow --pattern "src/checkout/**" --pattern "src/cart/**"
```

### `context update` failed with "topic not found"

**Cause:** The slug doesn't exist in this workspace.

**Solution:**
```bash
driftless context search <keyword>        # find a similar topic
driftless context list                    # browse everything
```
If genuinely missing, use `context add` instead of `update`.

### My `--gotcha` value got truncated / split

**Cause:** Shell quoting issue. The value contains a quote char or shell metacharacter.

**Solution:** Quote with single-quotes when the value contains double-quotes:
```bash
driftless context update billing-flow \
  --gotcha 'Stripe says "succeeded" before funds settle — wait for charge.updated'
```

Or use a heredoc-style file:
```bash
driftless context update billing-flow --content @path/to/note.md
```

---

## Install-skill / sync-skill

### `install-skill` says "AGENTS.md already configured" but the block looks stale

**Cause:** Older CLI versions wrote a block that doesn't match current markers. The new CLI self-heals by replacing the block between `<!-- driftless:start -->` and `<!-- driftless:end -->` on every run.

**Solution:** Upgrade the CLI and re-run:
```bash
npm install -g @driftless-sh/cli@latest
driftless install-skill
```

### `install-skill` downloads SKILL.md but `.driftless/assets/templates/` is empty

**Cause:** The asset download failed silently (best-effort) or the CLI is older than 0.2.4.

**Solution:**
```bash
npm install -g @driftless-sh/cli@latest
driftless install-skill                   # re-run; assets are re-fetched on every run
ls .driftless/assets/templates/           # verify
```

### `scripts/sync-skill.sh` says "Mirror already up to date — nothing to publish"

**Cause:** The canonical `skills/driftless/` directory in the monorepo matches the mirror. No diff to push.

**Solution:** Expected behavior. If you expected a change to publish, verify it landed in `skills/driftless/` (not somewhere else like `.driftless/`) and re-run.

---

## Context doctor

### `context doctor` reports `orphaned` topics

**Cause:** The topic references a repo id that no longer exists in Cloud (repo was deleted from the workspace).

**Solution:** Hidden from default agent reads. To resolve:
- Re-link the topic to a current repo: `driftless context link <slug>` (from the repo dir).
- Or delete the topic: `driftless context delete <slug>`.

### `context doctor` reports `repo_leak` topics

**Cause:** A topic's `where_repos` contains a repo id that doesn't exist in any workspace.

**Solution:** Investigate — this usually means a workspace migration left stale references. Contact support if widespread, or manually clean up via `context link` from a real repo.

### `context doctor` reports `zombie` topics

**Cause:** The topic's anchor patterns matched files when written but every pattern now globs to 0 files in this checkout — usually after a rename, a folder move, or a refactor that didn't update the topic.

**Solution:** Re-anchor or archive:
```bash
# Re-anchor to where the code lives now:
driftless context update <slug> \
  --remove-pattern "<old glob>" \
  --add-pattern    "<new glob>"

# Or if the concept is genuinely obsolete:
driftless context delete <slug>
```

### `context doctor` reports many `draft` topics

**Cause:** Suggested topics that were never promoted to `reviewed`.

**Solution:**
```bash
driftless context list --suggested
# for each suggested topic worth keeping:
driftless context update <slug> --what "..." --how "..." --status reviewed
# delete the rest:
driftless context delete <slug>
```

---

## When to escalate

If `driftless doctor` is all green but behavior is still wrong:
- `driftless doctor --json` and read the structured output.
- Check the dashboard (driftless.icu) — your view of topics may not match what Cloud has.
- File an issue at github.com/driftless-sh/driftless-skill/issues with the doctor JSON.
