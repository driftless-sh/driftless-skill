# Troubleshooting

Structured catalog of common errors. Format: **Symptom** → **Cause** → **Solution**.

For coverage status confusion, see `coverage-interpretation.md`. For topic field/kind questions, see `topic-anatomy.md`.

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

**Cause:** Driftless was never initialized for this repo, OR the repo is connected but no one has created topics yet (topic-first onboarding).

**Solution:** Don't assume the repo needs `driftless init`. Create your first topic:
```bash
driftless context add onboarding-context \
  --kind domain-map \
  --what "Shared context map for this repo." \
  --how "Captures what humans and agents need to know."
```

Ask the user before running `driftless init` — repo scan is optional enrichment.

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

### `context load --files "a.ts" --json` returns empty `topics`

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

## Graph & coverage

### `graph file <path>` returns empty `entrypoints`

**Cause (most common):** No HTTP route reaches this file. It's a pure utility, a library function, or an internal-only service.

**Cause (sometimes):** Scanner couldn't parse the file (low confidence). Components for the file are missing.

**Solution:**
- Verify with `--depth 4` to extend reachability.
- Open the file. If it's an entrypoint that should be detected (e.g. a NestJS `@Controller` or a Next.js page) but isn't, file an issue.
- If it's intentionally not an entrypoint, the empty list is correct.

### `graph file` returns `guards: []` AND `global_guards: []`

**Cause:** The scanner found no `@UseGuards`, no `APP_GUARD`, and no Clerk `auth()`/`currentUser()` call.

**Solution:** **DO NOT** assume the file is "public". The repo may use:
- A composed auth decorator the scanner doesn't recognize.
- A non-Clerk auth helper (custom middleware, Next.js middleware.ts).
- An API gateway / reverse proxy that authenticates before the request reaches the app.

Open the file and verify auth manually. If you confirm an auth pattern, document it as a topic (UC2) with an `invariant` so future agents see it.

### `graph coverage --json` shows everything as "gap"

**Cause:** Either no topics exist, or no topics have anchors (`patterns`).

**Solution:**
- `context list` — confirm topics exist.
- For each topic, check `anchors` via `context get <slug>`.
- Add anchors to topics that should claim files:
  ```bash
  driftless context update <slug> --add-pattern "src/billing/**"
  ```

---

## Context add / update

### `context add` rejected with "pattern matches 0 files locally"

**Cause:** The glob you passed doesn't match any file in the current scan root. Prevents creating a topic anchored to nothing.

**Solution:**
- Verify the glob is correct (try `ls src/billing/**` with shell glob expansion enabled).
- If the path is correct but the scanner hasn't seen it, run `driftless init` to refresh the baseline (ASK the user first if not initialized for this repo).

### `⚠ over-broad anchor: covers 187 components`

**Cause:** Your pattern covers more than 100 components — almost certainly catch-all.

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

### `context doctor` reports many `draft` topics

**Cause:** Someone ran `driftless init --suggest` and the auto-generated drafts were never promoted.

**Solution:**
```bash
driftless context list --suggested
# for each suggested topic:
driftless context update <slug> --what "..." --how "..." --status reviewed
```
Or delete the ones that aren't worth promoting.

---

## When to escalate

If `driftless doctor` is all green but behavior is still wrong:
- `driftless doctor --json` and read the structured output.
- Check the dashboard (driftless.icu) — your view of topics may not match what Cloud has.
- File an issue at github.com/driftless-sh/driftless-skill/issues with the doctor JSON.
