# Driftless Cloud Architecture

## What Cloud stores

Driftless Cloud is the **source of truth** for your team's codebase memory. It is NOT local session memory — it's shared, durable, and kept updated through scans and repo changes.

### Data stored in Cloud

| Entity | Description |
|--------|-------------|
| **Workspaces** | One per organization (maps to Clerk org) |
| **Repos** | Connected repositories per workspace |
| **Components** | Codebase component map (endpoints, guards, services, modules) |
| **Component Relations** | How components connect (uses_guard, depends_on, calls, imports) |
| **Architectural Intents** | What each repo/component is intended for |
| **Rules** | Structural and semantic rules defined by the team |
| **Watchers** | Context watchers — what, how, where for key concepts |
| **Watcher Events** | History of changes to watched files |
| **Context Events** | Per-component event history |
| **Violations** | Detected rule violations with diffs and explanations |
| **Integrations** | Email notifications (future: Slack) |
| **API Keys** | Per-workspace authentication keys for CLI/CI |

## How Cloud stays updated

### 1. Webhooks (continuous sync)
GitHub pushes → webhook → Cloud:
- Stores the diff
- Updates watchers when tracked files change
- Creates watcher_events and context_events

Webhooks are **not the product**. They are one sync signal that keeps Cloud memory fresh.

### 2. Agent scans (on-demand)
Agents run `driftless scan --diff`:
- Local diff sent to Cloud
- Evaluated against active rules
- Violations stored in Cloud
- Results returned to the agent

### 3. CLI init (initial setup)
`driftless init` connects a repo:
- Clones the repo
- Runs the scanner (3 passes: identity, component map, enrichment)
- Stores the baseline in Cloud
- Auto-suggests rules

## The Scanner — Three Passes

### Pass 1 — Identity (zero tokens)
Reads `package.json`, `tsconfig.json`, folder structure to detect:
- Framework (NestJS, Express, Next.js)
- System type (API, app, CLI, library)
- Language (v1: TypeScript only)
- Auth patterns (Clerk, Cognito, JWT, API key)

### Pass 2 — Component Map (zero tokens)
AST traversal using ts-morph:
- `@Controller`, `@Get`, `@Post` → endpoints
- `@UseGuards(Guard)` → guard relations
- `@Injectable()` → services
- `@Module()` → modules
- Extracts HTTP methods, paths, line numbers

### Pass 3 — Semantic Enrichment (minimal tokens)
For components without clear business meaning:
- Sends individual handler functions (max 100 lines) to BYOM
- Asks: "what domain does this expose? what actor consumes it?"
- Never sends the full repo
- Never more than 100 lines per call

## Auth Model

### Dual authentication

| Path | Auth Method | Use Case |
|------|-------------|----------|
| Dashboard | Clerk (JWT + orgId) | Browser-based management |
| CLI / CI | X-API-Key header | Agents, pipelines, scripts |
| Webhooks | HMAC-SHA256 | GitHub push events |

### API Key validation
```
CLI sends X-API-Key: drift_xxx
  → SHA-256 hash
  → Lookup in api_keys table
  → Resolve workspace_id
  → All queries filter by workspace_id
```

### Clerk org → workspace mapping
```
Browser → Clerk JWT → orgId
  → Lookup workspaces by clerk_org_id
  → Resolve workspace_id
  → URL :slug must match org slug
  → All queries filter by workspace_id
```

## Multi-tenancy

Each workspace is fully isolated:
- Separate rules
- Separate watchers
- Separate violations
- Separate repos and components
- Separate API keys

One Clerk organization = one Driftless workspace = one tenant.

## Tech Stack

- **API**: NestJS + TypeScript
- **Database**: PostgreSQL (Supabase)
- **Auth**: Clerk (organizations) + API keys
- **Scanner**: ts-morph (TypeScript AST)
- **Rules Engine**: Deterministic structural patterns + LLM-assisted semantic rules
- **CLI**: Node.js, published via npm
- **Dashboard**: React + Vite, deployed via Vercel
- **Infra**: Render (API), Vercel (dashboard), Supabase (DB)
