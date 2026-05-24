# Driftless — Agent Skill

Teaches AI agents (Claude Code, Codex, Cursor, and others) how to use Driftless to load team context before coding, persist what they learn back to Cloud, and refresh stale context before they push.

## Install

```bash
npx skills add driftless-sh/driftless-skill
```

The Driftless CLI also installs this skill automatically into any repo:

```bash
driftless install-skill    # writes CLAUDE.md, AGENTS.md, .driftless/{skill.md,references/*,assets/templates/*}
```

## What it does

Without this skill, every agent session starts from zero — re-deriving how auth works, what the billing constraints are, why a pattern was chosen. With it, agents load accumulated team knowledge before touching code and save what they learn back to Cloud.

The skill routes the agent through four use cases based on the current situation:

| Use case | When | First command |
|---|---|---|
| **UC0 — First-time setup** (topic-first) | Doctor fails or no topics exist | `driftless context add <slug>` (ask before running `driftless init` — repo scan is optional enrichment) |
| **UC1 — Load context** | Starting or resuming work, or about to edit a file | `driftless context load --files "<path>" --json` |
| **UC2 — Save discovery** | Learned a gotcha, decision, invariant, or constraint | `driftless context update <slug> --gotcha "…" --decision "…"` (batched) |
| **UC3 — Pre-commit refresh** | About to commit or push | `driftless sync && driftless context get --diff` |

The skill is designed around the principle that **Driftless informs, never blocks** — it is a context + drift-awareness layer, not a linter or PR gate.

## Requirements

- [Driftless CLI](https://www.npmjs.com/package/@driftless-sh/cli) installed and authenticated
- A Driftless workspace (free at [driftless.icu](https://driftless.icu))

```bash
npm install -g @driftless-sh/cli
driftless login --key <api-key>   # get a key at driftless.icu → Settings → API Keys
driftless install-skill            # writes the skill into the current repo
```

Topic-first means **no repo scan required to start.** Create your first topic with `driftless context add` and Driftless is useful immediately. Run `driftless init` later if you also want component discovery, code anchors, deterministic graph coverage, and stale-context detection — but ask the user first.

## Structure

```
skills/driftless/
├── SKILL.md                              # Main skill — situation routing + 4 Use Cases + examples
├── references/
│   ├── commands.md                       # Full CLI command reference
│   ├── workflow.md                       # The Driftless Loop in depth
│   ├── cloud.md                          # Cloud data model and architecture
│   ├── topic-anatomy.md                  # Kinds, fields, append-vs-replace, anchoring, [[wiki-links]], relations
│   ├── coverage-interpretation.md        # Decision tree for `context load` output
│   └── troubleshooting.md                # Symptom / Cause / Solution catalog
└── assets/
    └── templates/
        ├── code-context.md               # Scaffold for `--kind code-context` topics
        ├── decision.md                   # ADR-style scaffold for `--kind decision`
        ├── runbook.md                    # Operational scaffold for `--kind runbook`
        └── integration-note.md           # 3rd-party integration scaffold for `--kind integration-note`
```

When an agent creates a new topic, it uses the matching template:

```bash
driftless context add billing-flow \
  --kind code-context \
  --content @.driftless/assets/templates/code-context.md \
  --pattern "src/billing/**"
```

The YAML frontmatter in each template (`what`, `how`, `ownership`) is parsed automatically by the Driftless API and assigned to topic fields.

## Compatibility

| Surface | How it loads the skill |
|---|---|
| Claude Code | `CLAUDE.md` at repo root contains `@.driftless/skill.md`; Claude auto-loads it on session start |
| Codex / Cursor / other AGENTS.md readers | `AGENTS.md` at repo root contains the managed `<!-- driftless:start -->` block — a condensed workflow-first cheatsheet |
| Claude.ai (web) | Upload the skill via Settings → Capabilities → Skills |

## Links

- **Dashboard:** https://driftless.icu
- **Docs:** https://driftless.icu/docs
- **CLI on npm:** https://www.npmjs.com/package/@driftless-sh/cli
- **License:** MIT
