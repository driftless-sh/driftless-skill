# Driftless Cloud Architecture

## What Cloud Stores

Driftless Cloud is the **source of truth** for your team's codebase memory. It is not local session memory. It is shared, durable context kept updated through scans, repo activity, and agent updates.

### Data Stored In Cloud

| Entity | Description |
|--------|-------------|
| **Workspaces** | One per organization (maps to Clerk org) |
| **Repos** | Connected repositories per workspace |
| **Components** | Codebase component map (endpoints, guards, services, modules) |
| **Component Relations** | How components connect (uses_guard, depends_on, calls, imports) |
| **Watchers / Topics** | Context topics — what, how, where, gotchas, decisions |
| **Watcher Events** | History of changes to watched files |
| **Context Events** | Per-component event history |
| **Integrations** | GitHub App installation links |
| **API Keys** | Per-workspace authentication keys for CLI/agents |

## How Cloud Stays Updated

### 1. Webhooks

GitHub pushes and PR activity go to Cloud:
- Stores repo activity metadata
- Updates topics when tracked files change
- Creates watcher events and context events
- Powers PR context comments

Webhooks are one sync signal that keeps Cloud memory fresh.

### 2. CLI Init

`driftless init` connects a repo:
- Runs the scanner locally
- Builds the component baseline
- Uploads structural metadata to Cloud
- Suggests draft topics for modules
- Reports existing docs for intentional syncing

### 3. Agent Updates

Agents use `driftless context add`, `context update`, and `context sync` to persist durable discoveries. This is how team memory improves after real work.

## The Scanner

### Pass 1 — Identity

Reads `package.json`, `tsconfig.json`, folder structure:
- Framework (NestJS, Express, Next.js)
- System type (API, app, CLI, library)
- Language (v1: TypeScript only)
- Auth patterns (Clerk, Cognito, JWT, API key)

### Pass 2 — Component Map

AST traversal using ts-morph:
- `@Controller`, `@Get`, `@Post` -> endpoints
- `@UseGuards(Guard)` -> guard relations
- `@Injectable()` -> services
- `@Module()` -> modules
- Extracts HTTP methods, paths, line numbers

### Pass 3 — Context Coverage

Compares topics to components and files:
- Which files are strongly covered by reviewed topics
- Which areas only have broad or draft coverage
- Which high-risk files have no durable context

## Auth Model

| Path | Auth Method | Use Case |
|------|-------------|----------|
| Dashboard | Clerk (JWT + orgId) | Browser-based management |
| CLI | X-API-Key header | Agents and scripts |
| Webhooks | HMAC-SHA256 | GitHub events |

## Multi-Tenancy

Each workspace is fully isolated:
- Separate repos and components
- Separate topics
- Separate API keys
- Separate integrations

One Clerk organization = one Driftless workspace = one tenant.

## Tech Stack

- **API**: NestJS + TypeScript
- **Database**: PostgreSQL (Supabase)
- **Auth**: Clerk (organizations) + API keys
- **Scanner**: ts-morph (TypeScript AST)
- **CLI**: Node.js, published via npm
- **Dashboard**: React + Vite, deployed via Vercel
- **Infra**: Render (API), Vercel (dashboard), Supabase (DB)
