# Driftless — Agent Skill

Teaches AI agents (Claude Code, Codex, Cursor, and others) how to use Driftless to load team context before coding, persist discoveries, and verify integrity before pushing.

## Install

```bash
npx skills add driftless-sh/driftless-skill
```

## What it does

Without this skill, every agent session starts from zero — re-discovering how auth works, what the billing constraints are, why a pattern was chosen. With it, agents load accumulated team knowledge before touching code and save what they learn back to Cloud.

The skill routes the agent through four phases based on the current situation:

| Phase | When | Action |
|---|---|---|
| **UC0 — Init** | Repo has no Driftless context yet | `driftless init` |
| **UC1 — Load context** | Starting or resuming work | `driftless context get <slug>` |
| **UC2 — Save discovery** | Learned a gotcha or decision | `driftless context update <slug>` |
| **UC3 — Pre-push check** | About to commit | `driftless scan --diff` |

## Requirements

- [Driftless CLI](https://www.npmjs.com/package/@driftless-sh/cli) installed and authenticated
- A Driftless workspace with at least one connected repo

```bash
npm install -g @driftless-sh/cli
driftless login --key <api-key>   # get key at driftless.icu → Settings → API Keys
driftless init                     # run once per repo
```

## Structure

```
skills/driftless/
├── SKILL.md              # Main skill definition — loaded by agents
└── references/
    ├── commands.md       # Full CLI command reference
    ├── workflow.md       # Detailed workflow guide
    └── cloud.md          # Cloud architecture overview
```

## Links

- **Dashboard:** https://driftless.icu
- **Docs:** https://driftless.icu/docs
- **CLI on npm:** https://www.npmjs.com/package/@driftless-sh/cli
- **License:** MIT
