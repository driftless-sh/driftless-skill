# Coverage Interpretation — Decision Tree

`driftless context load --files "<path>" --json` returns, per file, a `coverage` block:

```json
{
  "status": "reviewed | partial | none",
  "match_specificity": "file | component | pattern | repo",
  "high_risk": true | false,
  "recommended_action": "document-now | add-topic | refresh-stale | reattach-or-remove | review-suggested | promote-draft | none"
}
```

**You MUST act on `coverage` — never just read it.** This file is the canonical decision tree.

## Step 1 — Read `status` + `match_specificity` together

`status` is *how solid* the covering topic is. `match_specificity` is *how tightly* it matches the file. Both must be strong for the context to be authoritative.

| `status` | `match_specificity` | Verdict |
|---|---|---|
| `reviewed` | `file` | **Authoritative.** Topic explicitly anchors this exact file. Proceed. |
| `reviewed` | `component` | **Authoritative.** Topic anchors the component this file belongs to. Proceed. |
| `reviewed` | `pattern` | **Loose match.** A broad pattern catches this file but the topic isn't *about* it. Verify against code. |
| `reviewed` | `repo` | **Repo-level only.** Topic mentions this repo but doesn't anchor at file/pattern level. Treat as background. |
| `partial` | any | **Some coverage, gaps remain.** Read the topic, then verify against current code. |
| `none` | — | **No coverage.** Read the code first. Create or anchor a topic (UC2) before assuming. |

**Rule:** only `reviewed` + (`file` OR `component`) means "the loaded context is authoritative." Anything else means **verify against code, then re-anchor (UC2)** — this is how context stays true to the repo over time.

## Step 2 — Check `high_risk`

`high_risk: true` means the file lives on an auth, security, or data-integrity surface (e.g. a guard, an auth middleware, a payments handler). The threshold for trust is higher.

| `high_risk` | Coverage | Action |
|---|---|---|
| `true` | strong (`reviewed` + `file`/`component`) | Proceed, but still verify auth invariants before changing behavior. |
| `true` | weak (anything else) | **Stop.** Read the code thoroughly. Document the invariant (UC2) BEFORE changing behavior. |
| `false` | strong | Proceed. |
| `false` | weak | Proceed with normal verification. |

**`high_risk: true` + weak coverage is the one case where you SHOULD NOT write code first.** Document the invariant, then change behavior.

## Step 3 — Follow `recommended_action` literally

The API does the math and tells you the next move. Follow it.

| `recommended_action` | Trigger | What to do |
|---|---|---|
| `none` | Coverage is solid (strong + reviewed) | Proceed with the task. |
| `document-now` | `high_risk: true` AND no strong reviewed coverage | UC2 — create a topic for this surface BEFORE changing behavior. |
| `add-topic` | `high_risk: false` AND `status: none` | UC2 — after understanding the area, add a topic. |
| `refresh-stale` | The covering topic is marked `stale` | UC2 — read current code, then update the topic. |
| `reattach-or-remove` | Only orphaned topics (repo was deleted) cover this file | Either re-link the topic to the current repo via `context link`, or remove it. |
| `review-suggested` | An auto-generated `suggested` topic covers this | Confirm, flesh out content, promote to `reviewed` (UC2). |
| `promote-draft` | A `draft` topic with usable content covers this | Verify content matches code, then promote to `reviewed` (UC2). |

## Worked examples

### Example A — High-risk surface, no context

```json
{
  "file": "src/auth/refresh-token.guard.ts",
  "coverage": {
    "status": "none",
    "match_specificity": "repo",
    "high_risk": true,
    "recommended_action": "document-now"
  }
}
```

**Action:** Do NOT touch the guard yet. Read the code, identify the invariant ("refresh tokens MUST expire after 15min idle"), create a topic:

```bash
driftless context add refresh-token-guard \
  --kind code-context \
  --invariant "Refresh tokens must expire after 15min idle" \
  --invariant "Token rotation must invalidate the previous token atomically" \
  --pattern "src/auth/refresh-token.guard.ts" \
  --tags auth,security
```

Then make the change.

### Example B — Strong coverage, low risk

```json
{
  "file": "src/billing/billing.service.ts",
  "coverage": {
    "status": "reviewed",
    "match_specificity": "file",
    "high_risk": false,
    "recommended_action": "none"
  }
}
```

**Action:** Read the matched topic via `context get billing` and proceed.

### Example C — Loose pattern match

```json
{
  "file": "src/utils/string-helpers.ts",
  "coverage": {
    "status": "reviewed",
    "match_specificity": "pattern",
    "high_risk": false,
    "recommended_action": "none"
  }
}
```

**Action:** The matching topic is anchored to `src/utils/**` but isn't really about string helpers. Read the code; if you learn something durable about string-helpers specifically, consider adding a narrower topic.

### Example D — Stale topic

```json
{
  "file": "src/checkout/checkout.controller.ts",
  "coverage": {
    "status": "partial",
    "match_specificity": "file",
    "high_risk": false,
    "recommended_action": "refresh-stale"
  }
}
```

**Action:** Get the topic, check `stale_reason`, compare against current code, then UC2:

```bash
driftless context get checkout-flow             # read stale_reason
# read the current code
driftless context update checkout-flow \
  --gotcha "Checkout now uses Stripe Payment Intents instead of Charges (migrated 2026-04)" \
  --decision "Switched to Payment Intents for SCA compliance and stronger auth flows"
```

The topic is now fresh; agents in future sessions get accurate context.

### Example E — Orphaned only

```json
{
  "file": "src/legacy/orders.service.ts",
  "coverage": {
    "status": "partial",
    "match_specificity": "pattern",
    "high_risk": false,
    "recommended_action": "reattach-or-remove"
  }
}
```

**Action:** The only topic covering this file references a repo that was deleted. Either:
- The topic is still relevant — link it to the current repo: `driftless context link legacy-orders`
- The topic is dead — `driftless context delete legacy-orders --dry-run` then delete.

## Anti-patterns

- **Treating `partial` as good enough.** It isn't. Partial means *some* coverage, *gaps remain*. The gap might be exactly the thing you're about to touch.
- **Treating `repo`-level match as actionable context.** It tells you the repo exists in Cloud, nothing more.
- **Ignoring `high_risk: true` because the file "looks simple".** The flag is set deterministically by the scanner based on auth surface. Trust it.
- **Skipping `recommended_action` because you "already know what the topic says".** The action accounts for staleness, draft status, orphan status — things you can't see from the topic body alone.

## Summary

```
1. status + match_specificity → is this authoritative?
2. high_risk → does this need extra verification?
3. recommended_action → what's the literal next step?
```

If any of those three says "verify" or "document-now", do that BEFORE writing the code change.
