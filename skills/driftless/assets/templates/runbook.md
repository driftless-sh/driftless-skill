---
what: "[One-line description of the operational scenario this runbook covers]"
ownership: "[On-call team or @handle]"
---

# [Runbook Title]

**Trigger:** [The exact alert, error, or condition that causes someone to open this runbook]
**Severity:** [P0 | P1 | P2 | P3]
**Estimated time to resolve:** [Range, e.g. 5–15 min]
**Last validated:** [YYYY-MM-DD]

## Symptoms

- [What the user or operator observes]
- [What dashboards or logs show]
- [What error message text to search for]

## Diagnosis

1. [First check — exact command, query, or dashboard URL to confirm the issue is what you think]
2. [Second check — narrow down the cause]
3. [Decision point — which Path below applies]

## Resolution

### Path A — [Most common cause]

1. [Step — exact command or action]
2. [Step]
3. **Verification:** [How to confirm it worked — command output, dashboard metric, log line]

### Path B — [Alternate cause]

1. [Step]
2. [Step]
3. **Verification:** [...]

## Rollback

[If the resolution makes things worse, how to revert. Include the exact command or rollback procedure.]

## Verification (post-resolution)

- [Command or dashboard that confirms the system is healthy again]
- [Metric or log line that should return to normal — include the expected value or pattern]

## Post-incident

- [What to capture in the incident write-up — link to [[incident-template-slug]] if applicable]
- [Whether this runbook needs updating based on what was learned — common case is adding a new Path]

## Related

- [[other-topic-slug]] — [why related, e.g. the reference topic for the affected module]
