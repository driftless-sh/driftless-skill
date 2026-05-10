# Driftless — Agent Skill

Shared codebase memory layer for AI engineering teams.

## Install

```bash
npx skills add joseluistello/driftless-skill
```

Or manually clone into your skills directory.

## What it does

Teaches AI agents (Claude Code, Codex, Cursor) to:

1. **Load context** before coding — `driftless context get <feature>`
2. **Scan diffs** before pushing — `driftless scan --diff`
3. **Document discoveries** — `driftless context add "name" --what "..."`

Agents follow a strict loop: load → implement → scan → fix → push.

## Quick start

```bash
npm install -g @driftless-sh/cli
driftless login --key drift_xxx
driftless init
```

## Structure

```
skills/driftless/
├── SKILL.md                  # Main skill definition (loaded by agents)
└── references/
    ├── commands.md           # CLI command reference
    ├── workflow.md           # Agent workflow guide
    └── cloud.md              # Cloud architecture overview
```

## Links

- **Docs:** https://driftless.icu/docs
- **API:** https://api.driftless.icu/api/v1
- **CLI:** https://www.npmjs.com/package/@driftless-sh/cli
- **License:** MIT
