# Driftless CLI — Command Reference

**Version:** 0.1.12  
**Install:** `npm install -g @driftless-sh/cli`

---

## Auth

```bash
driftless login --key drift_xxx
```

Or via env vars: `DRIFTLESS_API_KEY`, `DRIFTLESS_API_URL` (default: `https://api.driftless.icu/api/v1`).

Config saved to `~/.driftless/config.json`.

---

## Commands

### `driftless init`

Connect repo to Cloud, scan codebase, upload baseline, suggest rules, install `AGENTS.md`.

```bash
driftless init
```

Must run from git repo root with a remote configured.

### `driftless scan`

Evaluate local changes against Cloud rules.

```bash
driftless scan           # staged + uncommitted
driftless scan --diff    # uncommitted only
```

Returns: `{ "status": "clean" | "violated", "violations": [...] }`

### `driftless context`

Manage context watchers. JSON output by default, `--human` for readable text.

| Subcommand | Description |
|---|---|
| `list` | List all watchers |
| `get <slug>` | Get watcher with components + history |
| `add <slug>` | Create watcher (see fields below) |
| `update <slug>` | Update watcher fields |
| `delete <slug>` | Delete watcher |
| `search <query>` | Full-text search (what, how, decisions, gotchas) |

**Fields for add/update:**

| Flag | Description |
|---|---|
| `--what` | What this piece is |
| `--how` | How it works |
| `--pattern` | Glob for dynamic resolution (e.g. `src/sdk/**`) |
| `--where` | Explicit file path (alternative to pattern) |
| `--decisions` | Design rationale |
| `--gotchas` | Known edge cases |
| `--ownership` | Who maintains it |

**Examples:**
```bash
driftless context add "b2b-guard" \
  --what "Guard for B2B endpoints" \
  --pattern "src/shared/guards/**" \
  --decisions "Multi-tenant via org_id" \
  --gotchas "Fails with Clerk test tokens"

driftless context get b2b-guard --human
driftless context search "auth"
driftless context update b2b-guard --gotchas "New edge case"
driftless context delete old-feature
```

### `driftless install-skill`

Install `AGENTS.md` into current repo with Driftless instructions for AI agents.

```bash
driftless install-skill
```

### Help

```bash
driftless --help
driftless help <command>
driftless --version
```
