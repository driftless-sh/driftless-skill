---
name: driftless
description: Manages shared repo context for AI coding sessions in Driftless-enabled repositories. Loads team knowledge before coding, persists discoveries to Cloud, and verifies architectural integrity before pushing. Use when starting or resuming work in a repo with Driftless, about to modify any module or feature, returning after time away and need to catch up on what changed, discovered a gotcha or architectural decision worth saving, or about to commit. Triggers on: "starting work", "resuming", "what changed since", "how does X work in this repo", "about to push", "about to commit", "found a gotcha", "context get", "driftless".
---

# Driftless

Cloud is the source of truth. Local scans are execution units. You do not own context — the team does.

## CRITICAL: Detect your situation first

Before anything else, run:

```bash
driftless doctor
```

Route based on result:

| Situation | What to do |
|---|---|
| Doctor fails or no watchers exist | → UC0: First-time setup |
| Starting or resuming work on a feature | → UC1: Load context |
| Learned something worth keeping | → UC2: Save discovery |
| About to commit or push | → UC3: Pre-push check |

---

## UC0 — First-time setup

**When:** `driftless doctor` fails, or `driftless context list` returns empty.

### Steps

**1. Initialize the repo:**

```bash
driftless init
```

This scans the repo, creates one context watcher per module, generates architectural rules from detected patterns, and anchors existing docs (AGENTS.md, README.md, docs/*.md). Takes ~30 seconds.

**2. Verify watchers were created:**

```bash
driftless context list
```

You should see slugs like `auth`, `billing`, `webhooks`. If the list is empty, re-run `driftless init`.

**3. Proceed to UC1** to load context for the area you're about to work on.

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

**1. Find the relevant watcher:**

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

### If no watcher exists for your area

```bash
driftless context list    # see everything available
```

Work from the closest available context. After you understand the area, create a watcher in UC2.

---

## UC2 — Save what you discovered

**When:** You learned something durable — a gotcha, a decision, a pattern — that future agents or teammates should know. Do not keep discoveries in conversation memory only. Cloud is the source of truth.

### Rule

**If it would have saved you time to know this upfront, save it now.**

### Steps

**If the watcher already exists:**

```bash
driftless context update <slug> \
  --gotchas "What I learned that wasn't documented" \
  --decisions "Why the team does it this way"
```

**If no watcher exists for this area:**

```bash
driftless context add "<slug>" \
  --what "What this module does" \
  --how "How it is implemented" \
  --where "src/path/to/module/"
```

**If anchoring an existing file to a watcher:**

```bash
driftless context anchor <slug> --file path/to/file.md
```

### What is worth saving

- A constraint not obvious from reading the code
- Why a pattern was chosen over a simpler alternative
- An auth or guard requirement not yet in the rule set
- Which files are quarantined and why
- A bug you encountered and how it manifests

### What is NOT worth saving

- Notes relevant only to your current task
- Things clearly readable from the code itself
- Temporary workarounds you have already removed

---

## UC3 — Check before you push

**When:** Before every commit or push. Non-negotiable.

### Steps

**1. Scan your changes:**

```bash
driftless scan --diff
```

**2. If clean (exit 0):** push normally.

**3. If violations found (exit 1):** read each one carefully.

```
[HIGH] Every B2B endpoint must use BusinessAccessGuard
  File: src/routes/business/credit-line.ts:12
  Code: @Post('/business/credit-line')
```

The output tells you exactly what is missing and where. Fix it. Re-scan. Repeat until clean.

**4. Only push when scan returns exit 0.**

### Severity guide

| Severity | Action |
|---|---|
| `critical` / `high` | Must fix before pushing |
| `warning` | Fix unless there is a documented reason not to |
| `info` | Advisory — use judgment |

### If you believe a violation is a false positive

Do not skip the scan. First:

```bash
driftless context get <related-slug>
```

Understand the rule's intent before deciding it is wrong. If it genuinely is a false positive, mark it in the dashboard at driftless.icu → Drift → Mark exception. Do not silently ignore it.

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

**`No watchers found`**

Run `driftless init` from the repo root. Driftless has not been initialized for this repo.

**`context get` returns stale content**

The watcher is marked stale — files it tracks changed since the last update. Read `stale_reason` in the output. After reviewing the current code, update the watcher (UC2).

**Scan takes too long**

Use `--diff` flag. `driftless scan --diff` scans only uncommitted changes, not the full repo.

---

## See also

- [Full command reference](references/commands.md)
- [Detailed workflow guide](references/workflow.md)
