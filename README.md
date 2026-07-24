# spelflow-skills

Claude Code skills for working with [Spelflow Tracker](https://spelflow.ru).

## Skills

| Skill | Trigger | What it does |
|---|---|---|
| `spelflow-loader` | `/spelflow-loader` | General Spelflow Tracker REST API client — create/read Workspaces, Projects, Milestones, Issues; sprint-markdown and WWA-block input adapters; bridges to the pm-skills marketplace when the user only has a raw idea |
| `spelflow-plan` | `/spelflow-plan` | Orchestrates the full pm-skills pipeline for one epic: PRD → optional risk-check → per-feature WWA backlog → Spelflow push via `spelflow-loader` |

## Install

### Claude Code plugin marketplace

```
/plugin marketplace add BabylenMagnus/spelflow-skills
/plugin install spelflow-skills@spelflow-skills
```

### Any agent supported by `npx skills` (vercel-labs/skills)

```bash
npx skills add BabylenMagnus/spelflow-skills
```

Installs into whichever supported coding agent you're running (Claude Code, Cursor, OpenCode, Kilo Code, and others) — see [agentskills.io](https://agentskills.io) for the full agent list.

## Layout

```
skills/
  spelflow-loader/SKILL.md
  spelflow-plan/SKILL.md
.claude-plugin/
  marketplace.json
  plugin.json
```

Standard `skills/<name>/SKILL.md` layout, compatible with both the Claude Code plugin marketplace format and the `npx skills` discovery contract.
