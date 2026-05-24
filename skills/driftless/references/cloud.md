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
| **Topics** | Context topics — what, how, where, gotchas, decisions |
| **Topic Activity** | History of changes to a topic's covered files |
| **Context Events** | Per-component event history |
| **Integrations** | GitHub App installation links |
| **Pending Installations** | App installs awaiting a repo to link to — linking is order-independent |
| **API Keys** | Per-workspace authentication keys for CLI/agents |

## How Cloud Stays Updated

### 1. Webhooks

GitHub pushes and PR activity go to Cloud:
- Stores repo activity metadata (all branches, for audit)
- Marks a topic **stale** when covered code changes on a **tracked branch** (the default branch is always tracked; see `driftless branches`)
- Creates activity + context events
- Posts a short **nudge** comment on PRs that touch a topic (a heads-up, not a linter — it never blocks the PR)

Webhooks are one sync signal that keeps Cloud memory fresh.

### 2. CLI Init

`driftless init` enriches a repo with structural metadata:
- Runs the scanner locally (no git clone, structural metadata only — never source)
- Builds the component baseline
- Uploads structural metadata to Cloud
- **Only with `--suggest`**: suggests draft topics for modules (off by default)
- Reports existing docs for intentional syncing
- Reconciles a pending GitHub App install for this repo's org, if any (order-independent linking)

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

AST traversal using the TypeScript Compiler API (`ts.createSourceFile` per file — no global Project, streaming, no OOM):
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
- **Scanner**: TypeScript Compiler API (`ts.createSourceFile`, streaming, no OOM)
- **CLI**: Node.js, published via npm
- **Dashboard**: React + Vite, deployed via Vercel
- **Infra**: Render (API), Vercel (dashboard), Supabase (DB)
