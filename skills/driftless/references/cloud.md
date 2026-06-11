# Driftless Cloud Architecture

## What Cloud Stores

Driftless Cloud is the source of truth for the team's durable context. It is not local session memory. It stores topics, anchors, relations, repo links, PR activity, and audit history so humans and agents can retrieve the right context before touching files.

| Entity | Description |
|---|---|
| **Workspaces** | One per organization or team |
| **Repos** | Connected repositories per workspace |
| **Topics** | Durable context: what, how, gotchas, decisions, invariants, checks |
| **Topic Anchors** | `patterns`, `where_files`, and `where_repos` that connect topics to files/repos |
| **Topic Relations** | Typed edges between topics: depends_on, relates_to, supersedes, blocks, implements, documents, risk_for |
| **Topic Activity** | History of topic changes and file drift signals |
| **PR Observations** | Pull request metadata, matched topics, covered files, and uncovered file gaps |
| **Integrations** | GitHub App installation links and delivery settings |
| **Pending Installations** | GitHub installs awaiting a repo/workspace link |
| **API Keys** | Per-workspace authentication keys for CLI/agents |

Cloud stores topic text and metadata. It does not need source-code bodies to deliver context.

## How Cloud Stays Updated

### 1. Topic Writes

Humans and agents create and update topics:

```bash
driftless context add <slug> --what "..." --how "..." --pattern "src/auth/**"
driftless context update <slug> --gotcha "..." --decision "..."
```

This is the primary growth path. If a session reveals durable knowledge, append it to the matching topic before finishing.

### 2. Anchors

Anchors make topics show up at the moment of work:

- `patterns` match repository-relative globs.
- `where_files` match explicit files.
- `where_repos` records which repos use a topic.

The CLI validates patterns against the local checkout before writing. A zero-match pattern is blocked; an over-broad pattern is warned.

### 3. GitHub PR Activity

GitHub pushes and PR activity go to Cloud:

- Stores repo activity metadata for audit (every PR↔topic match is recorded, comment or not).
- Marks a topic stale when anchored files change on a tracked branch.
- Queues the Auditor on PRs that touch documented areas; the Auditor posts ONE comment only when it has a finding, carrying the affected topics' recorded context.
- Gap-finding lives in the coverage map (MCP `driftless_context_coverage` / dashboard), not in PR comments.

The Auditor informs; it never blocks. A clean PR is silent.

### 4. Agent Refresh

Agents use:

```bash
driftless sync
driftless context get --files "src/path/file.ts"
driftless context get --diff
```

`sync` pulls drift and PR signals. `context get --files` retrieves topic context before editing. `context get --diff` retrieves topics touched by local uncommitted changes before finishing.

## Auth Model

| Surface | Auth Method | Use Case |
|---|---|---|
| Dashboard | Clerk session + organization | Browser management |
| CLI | `X-API-Key` | Agents, scripts, local dev |
| MCP | OAuth bearer or API key | ChatGPT/Claude/local MCP clients |
| Webhooks | HMAC-SHA256 | GitHub events |

## Multi-Tenancy

Each workspace is isolated:

- Separate repos
- Separate topics
- Separate topic relations
- Separate API keys
- Separate integrations

One organization maps to one Driftless workspace.

## Tech Stack

- **API**: NestJS + TypeScript
- **Database**: PostgreSQL via Supabase
- **Auth**: Clerk organizations + encrypted API keys + OAuth for MCP
- **CLI**: Node.js, published via npm
- **Dashboard**: React + Vite
- **MCP**: protocol adapter that calls the REST API
- **Infra**: Render, Vercel, Supabase
