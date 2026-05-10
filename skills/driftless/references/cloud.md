# Driftless Cloud Architecture

## What Cloud stores

| Entity | Description |
|---|---|
| **Workspaces** | One per organization (Clerk org mapping) |
| **Repos** | Connected repositories per workspace |
| **Components** | Codebase map: endpoints, guards, services, modules |
| **Relations** | Connections: uses_guard, depends_on, calls, imports |
| **Intents** | What each repo is intended for |
| **Rules** | Structural + semantic rules (team-defined) |
| **Watchers** | Context: what, how, where, decisions, gotchas |
| **Violations** | Detected rule violations with diffs |
| **API Keys** | Per-workspace keys for CLI/CI auth |

## How Cloud stays updated

1. **Webhooks** — GitHub push → stores diff, updates watchers on tracked file changes
2. **Agent scans** — `driftless scan --diff` → evaluates against rules, stores violations
3. **CLI init** — Full scan (3 passes) → baseline + components + suggestions

## Scanner — 3 passes

| Pass | Method | Detects |
|---|---|---|
| **1. Identity** | Zero tokens (package.json, tsconfig, structure) | Framework, system type, auth patterns |
| **2. Component map** | Zero tokens (ts-morph AST) | Endpoints, guards, services, modules, relations |
| **3. Enrichment** | Minimal tokens (max 100 lines per call) | Domain semantics for unclear components |

Never sends full repo to LLM. Never more than 100 lines per call.

## Auth model

| Path | Method | Use case |
|---|---|---|
| Dashboard | Clerk (JWT + orgId) | Browser management |
| CLI / CI | `X-API-Key` header | Agents, pipelines |
| Webhooks | HMAC-SHA256 | GitHub push events |

API key flow: `X-API-Key` → SHA-256 hash → lookup → resolve `workspace_id` → all queries filter by workspace.

Clerk flow: JWT → `orgId` → lookup by `clerk_org_id` → resolve `workspace_id` → URL `:slug` must match.

## Multi-tenancy

One Clerk org = one workspace = one tenant. Fully isolated: rules, watchers, violations, repos, components, API keys.

## Tech stack

- **API:** NestJS + TypeScript (Render)
- **DB:** PostgreSQL via Supabase
- **Auth:** Clerk (orgs) + API keys
- **Scanner:** ts-morph (TypeScript AST)
- **CLI:** Node.js, npm package
- **Dashboard:** React + Vite (Vercel)
