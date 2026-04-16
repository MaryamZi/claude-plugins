---
name: run
description: Parse a Claude Code Insights report and generate or improve skills, agents, hooks, and CLAUDE.md rules based on actual usage patterns. Use when the user wants to turn their insights report into actionable automations.
tools: Read, Write, Edit, Bash
---

# Insights to Automations

Turn a Claude Code Insights report into concrete, installable automations — skills, agents, hooks, and CLAUDE.md rules — based on the user's actual behavioral patterns.

## Input

The insights report path. Defaults to `~/.claude/usage-data/report.html` if not specified via `$ARGUMENTS`.

## Core Principle

**Edit over create.** Before generating anything new, check what already exists in `.claude/skills/`, `.claude/agents/`, and `.claude/settings.json`. If a relevant automation already exists, improve it based on the insights rather than creating a duplicate.

## Workflow

### Phase 1: Parse the Insights Report

First, check if the report file exists at the specified path (or the default `~/.claude/usage-data/report.html`). If it doesn't exist:

1. Tell the user: "No insights report found. Run `/insights` first to generate one, then re-run `/insights-to-automations:run`."
2. Stop — do not proceed with the remaining phases.

If the report exists, it will be large (typically 500+ lines of HTML). **Do not try to read it all at once or re-read it multiple times.** Instead, read it in sequential chunks (e.g., 200 lines at a time using offset/limit) and extract data as you go. Read each chunk exactly once — do not re-read lines you've already seen.

Extract structured data from these sections as you encounter them. The table below lists known landmarks as **hints**, not hard requirements — the report format may evolve. Use landmarks when present, but fall back to heading text, section structure, or content semantics to identify sections. If a section can't be found at all, skip it gracefully and work with what you have.

| Section | Likely Landmarks (hints) | What to Extract |
|---------|--------------------------|-----------------|
| Project areas | `id="section-work"`, `.project-area` | Area names, session counts, descriptions |
| Usage stats | `.stats-row`, `.chart-card` | Languages, tools, session types, task types |
| Friction points | `id="section-friction"`, `.friction-category` | Friction titles, descriptions, examples |
| Wins | `id="section-wins"`, `.big-win` | What's working well (don't break these) |
| Suggested features | `id="section-features"`, `.feature-card` | Already-identified automation opportunities |
| CLAUDE.md suggestions | `.claude-md-section`, `.claude-md-item` | Suggested rules with rationale |
| New patterns | `id="section-patterns"`, `.pattern-card` | Workflow patterns with example prompts |
| Horizon ideas | `id="section-horizon"`, `.horizon-card` | Ambitious future workflows |
| At a glance | `.at-a-glance` | Summary of what's working, what's hindering, quick wins |

The minimum viable parse is **friction points** and **suggested features** — those two sections carry the most actionable signal. If you can only find those, that's enough to proceed.

### Phase 2: Inventory Existing Automations

Read these specific paths to inventory what exists. Do not glob or search broadly — check exactly these locations:

1. **Skills**: List `~/.claude/skills/` and read each `SKILL.md` found one level down (e.g., `~/.claude/skills/*/SKILL.md`)
2. **Agents**: List `~/.claude/agents/` and read each `.md` file found (e.g., `~/.claude/agents/*.md`)
3. **Hooks/settings**: Read `~/.claude/settings.json` — look at the `hooks` key for existing hooks
4. **User-level CLAUDE.md**: Read `~/.claude/CLAUDE.md` if it exists
5. **Project-level CLAUDE.md**: Read `./CLAUDE.md` in the current working directory if it exists
6. **Project-level .claude/**: Read `./.claude/settings.json`, `./.claude/CLAUDE.md` in the current working directory if they exist
If any path doesn't exist, skip it — don't error or search for alternatives.

**Do not check for CLI tools in this phase.** Only check for a specific CLI tool (e.g., `which gh`) later in Phase 3, at the point where you're about to recommend an MCP server that it would overlap with. Don't speculatively scan for tools upfront.

### Phase 3: Generate Automations

If Phase 1 produced no actionable patterns (no friction points, no recurring tasks, no suggested features), tell the user: "Your report doesn't surface patterns that would benefit from automation right now. This usually means things are working well as-is. Run `/insights-to-automations:run` again after your next batch of sessions." Then stop — do not proceed to Phase 4 or 5.

Otherwise, for each actionable pattern, decide the automation type and whether to create or edit.

#### Decision Matrix

| Signal from Report | Automation Type | Confidence |
|--------------------|----------------|------------|
| Recurring task type with 3+ sessions (e.g., "doc consistency sweeps") | **Skill** | High |
| Friction: "excessive changes" or "misunderstood scope" | **CLAUDE.md rule** | High |
| Friction: "wrong approach" on specific task type | **Skill** with guardrails | Medium |
| Suggested feature: hooks | **Hook** in settings.json | High |
| Suggested feature: MCP server | **MCP recommendation** (don't install) | Medium |
| Horizon: parallel agent workflow | **Agent** template | Medium |
| Horizon: autonomous iteration | **Skill** with agent delegation | Medium |
| Language/tool with high edit count but no lint hook | **Hook** | High |

#### Confidence Tiers

- **High confidence**: Hooks for detected linters/formatters, CLAUDE.md rules from friction patterns, simple skills for clearly repetitive tasks
- **Medium confidence**: Skills for complex workflows, agent templates, MCP recommendations
- **Low confidence**: Architecture changes, workflow restructuring, things that need user judgment

**All tiers require user approval before writing any files.** Never auto-install. Present the full contents, get explicit confirmation, then write.

### Phase 4: Check for Conflicts and Overlaps

Before writing any file:

1. **Existing skill with similar name or purpose?** → Propose edits to that skill instead
2. **Existing hook covering the same event?** → Propose extending/modifying, not duplicating
3. **Existing CLAUDE.md rule covering the same concern?** → Propose strengthening, not duplicating
4. **Would this automation conflict with the user's stated preferences?** → Flag and skip

### Phase 5: Output

Present results in two steps: overview first, then one-by-one walkthrough.

#### Step 1: Overview

Show a numbered summary table of all proposed automations across all confidence tiers:

```
| # | Type | Name | Confidence | One-line why |
|---|------|------|------------|--------------|
| 1 | CLAUDE.md | Scope rules | High | Friction: excessive changes (16 instances) |
| 2 | Skill | /scoped-rename | High | 14 doc sessions with recurring rename work |
| 3 | Hook | TS type-check | Medium | TS compilation errors caught late |
| ... | ... | ... | ... | ... |
```

After the table, ask the user: "Want to walk through these one by one, or skip any upfront?"

The user may:
- Proceed with all ("go ahead")
- Skip specific numbers ("skip 3 and 5")
- Stop entirely ("none of these")

#### Step 2: One-by-One Walkthrough

For each non-skipped automation, present its full details and wait for approval before moving to the next:

**High confidence items** — show:
- **Why**: The specific insight that motivated it (quote the report)
- **File**: Full path and contents
- **Existing?**: Whether this is new or an edit to an existing file

**Medium confidence items** — show:
- **Why**: The insight that motivated it
- **Draft**: A starting point that needs user input
- **Questions**: What the user should decide before installing

**Low confidence items** — show:
- Brief recommendation with next steps. Don't generate files.

After each item, ask: "Install this? (yes / no / modify)"
- **yes**: Write the file immediately
- **no**: Skip it
- **modify**: User describes changes, you revise and re-present

Only move to the next item after the current one is resolved.

#### Naming Convention

Prefix all generated automation names with `i2a-` (insights-to-automations) so they're easily distinguishable from hand-built ones:

- Skills: `i2a-scoped-rename`, `i2a-review`
- Agents: `i2a-doc-checker`
- CLAUDE.md sections: `## i2a: Scope Rules`

This makes it easy to list, find, or clean up generated automations later. If the user removes the prefix when approving, that's fine.

## Templates

Use the templates in this skill's directory as the starting structure for each automation type. Fill in the `{placeholders}` with values derived from the insights report.

- **Skill**: [templates/skill.md](templates/skill.md)
- **Agent**: [templates/agent.md](templates/agent.md)
- **Hook**: [templates/hook.json](templates/hook.json)
- **CLAUDE.md rule**: [templates/claude-md-rule.md](templates/claude-md-rule.md)

**Read templates lazily** — only read a template file when you're about to generate that specific automation type. If the report only produces CLAUDE.md rules and a hook, don't read the skill or agent templates at all.

Also reference [references/automation-schemas.md](references/automation-schemas.md) for full schema details (frontmatter fields, event types, exit codes, dynamic context injection, etc.). Only read this file if you need to look up a specific schema detail — don't read it preemptively.

## Important Guidelines

- **Fewer is better.** Aim for 3-5 high-value automations, not 10+ mediocre ones. The user can always run this again later.
- **Generic over specific.** If a pattern applies to docs AND code (e.g., scoped rename, consistency review), make the automation work for both — don't create a doc-specific version when a general-purpose one covers the same need. A `/scoped-rename` beats a `/doc-rename`.
- **No overlapping automations.** Before proposing a new automation, check if it overlaps with another one you're already proposing. If two automations cover similar ground, merge them into one or drop the weaker one. For example, don't propose both a rename skill and a parallel-rename agent — the skill is enough until proven insufficient.
- **Don't create agents prematurely.** Only propose an agent when the task genuinely needs parallel execution or isolated context. A skill handles most workflows fine. If the user outgrows a skill, they can upgrade to an agent later.
- **Respect what works.** The "wins" section shows patterns that are already effective — don't automate away things the user does well manually.
- **Address friction first.** The highest-value automations eliminate recurring pain points.
- **Be specific in instructions, generic in scope.** Skill instructions should be precise ("use Grep to find, confirm with user, then Edit"), but the skill itself should work across file types and projects.
- **Include scope guards.** Every skill should have explicit "do not" rules informed by the friction patterns (e.g., "do not modify files outside the specified directory").
- **For hooks, verify the tool exists.** Before recommending a lint hook, check that the linter is actually installed/configured in the project.
- **MCP servers are recommendations only.** Never auto-install MCP servers — just recommend with install command. Also skip if the user already has a CLI tool that covers the same functionality (e.g., `gh` for GitHub).
- **Reference the report.** When presenting automations, quote the specific insight that motivated each one so the user can evaluate relevance.
