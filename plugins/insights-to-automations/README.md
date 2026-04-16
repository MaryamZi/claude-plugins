# i2a — Insights to Automations

Turn your [Claude Code Insights](https://docs.anthropic.com/en/docs/claude-code) report into concrete, installable automations — skills, agents, hooks, and CLAUDE.md rules — based on your actual usage patterns.

## Prerequisites

Run `/insights` first to generate a report at `~/.claude/usage-data/report.html`.

## Usage

```
/insights-to-automations:run
```

Or with a custom report path:

```
/insights-to-automations:run /path/to/report.html
```

## What it does

1. Parses your insights report (friction points, recurring tasks, suggested features)
2. Inventories your existing automations (skills, agents, hooks, CLAUDE.md rules)
3. Proposes 3-5 high-value automations — editing existing ones where possible
4. Walks you through each proposal one by one for approval

All generated automations are prefixed with `i2a-` so they're easy to identify and clean up.

## What it generates

| Signal | Automation |
|--------|-----------|
| Recurring task (3+ sessions) | Skill |
| Friction: scope/approach issues | CLAUDE.md rule |
| Suggested hook | Hook in settings.json |
| Suggested MCP server | Recommendation (not auto-installed) |
| Horizon: parallel workflow | Agent template |
