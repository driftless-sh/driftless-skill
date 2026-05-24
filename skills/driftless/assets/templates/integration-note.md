---
what: "[What 3rd-party service this integration covers and what capability it provides us]"
how: "[Integration pattern — webhook, REST, GraphQL, SDK, message queue, polling]"
ownership: "[Team that owns the integration]"
---

# [Integration Name]

**Vendor:** [Service name + link to their docs]
**Auth method:** [OAuth, API key, mTLS, JWT, signed webhook, etc.]
**Rate limits:** [Numbers or link to vendor docs]
**Environment:** [Sandbox URL, Production URL]

## What we use it for

[The business capability this integration provides — be specific. "Stripe for payment processing" is too vague; "Stripe for one-time charges, subscriptions, and customer portal" is right.]

## How it's wired

[Architectural overview: which services call it, which DTOs serialize/deserialize, retry/idempotency strategy. Anchor the implementation detail to [[code-context-topic-slug]] rather than duplicating it here.]

## Auth & secrets

- **Credentials location:** [Env var name, secret manager path, .env file (never check in)]
- **Rotation policy:** [How often, who's responsible]
- **Last rotation:** [YYYY-MM-DD]
- **Access list:** [Who can read/rotate the secret]

## Gotchas

- [Quirk of the vendor's API that surprised us — be specific so future agents recognize it]
- [Timing, race, or ordering issue we hit]
- [Rate-limiting behavior we triggered in production]
- [Idempotency or duplicate-delivery behavior]

## Failure modes

- **Vendor outage:** [What happens to user-facing flows — degraded mode, queue + retry, hard failure]
- **Network failure / timeout:** [Retry strategy — exponential backoff, max attempts]
- **Auth failure (401/403):** [Likely cause — expired token, rotated secret, scope change]
- **Alerting:** [Where the alert goes — link to [[runbook-slug]] for response]

## Compliance / data flow

- **Data leaving our system to the vendor:** [List fields — PII, financial, health, etc.]
- **Data the vendor sends us:** [List]
- **Retention on the vendor side:** [Their policy + our DPA]
- **PCI / HIPAA / GDPR considerations:** [If applicable]

## Related

- [[code-context-slug]] — Implementation in the codebase
- [[runbook-slug]] — Recovery procedure when the integration is down
- [[decision-slug]] — Why we chose this vendor over alternatives
