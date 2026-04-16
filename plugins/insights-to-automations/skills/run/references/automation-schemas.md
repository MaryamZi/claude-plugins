# Automation Schemas Reference

Quick reference for generating each automation type.

## Skills

Location: `.claude/skills/<name>/SKILL.md`

Frontmatter fields:
- `name` (required): kebab-case identifier
- `description` (required): when to trigger — be specific
- `disable-model-invocation: true`: user-only (for side effects like deploy, commit, send)
- `user-invocable: false`: Claude-only (background knowledge)
- `allowed-tools`: restrict tool access (e.g., `Read, Grep, Glob` for read-only)
- `context: fork`: run in isolated sub-agent
- `agent: Explore`: agent type when forked
- `tools`: additional tools beyond defaults

Dynamic context with `!`command``:
```
## Current State
- Branch: !`git branch --show-current`
- Status: !`git status --short`
```

Arguments via `$ARGUMENTS`:
```
Rename all references of $ARGUMENTS across the project.
```

Skills can include supporting files:
```
.claude/skills/my-skill/
  SKILL.md
  templates/
  scripts/
  examples/
  references/
```

## Agents

Location: `.claude/agents/<name>.md`

Frontmatter fields:
- `model`: haiku, sonnet, or opus
- `allowed-tools`: tool access list

Model selection:
- **haiku**: simple, repetitive checks
- **sonnet**: most review/analysis (default)
- **opus**: complex reasoning, architecture

## Hooks

Location: `.claude/settings.json` under `hooks` key

Event types:
- `PreToolUse`: runs before a tool, can block with exit code 2
- `PostToolUse`: runs after a tool completes
- `Notification`: runs on notification events

Structure:
```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "shell command here"
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "shell command here"
          }
        ]
      }
    ]
  }
}
```

Hook commands receive tool input as JSON on stdin.
Environment variable `$CLAUDE_FILE` contains the file path for file-editing tools.

Exit codes for PreToolUse:
- 0: allow
- 2: block (with stderr as reason)

## CLAUDE.md Rules

Location: `CLAUDE.md` (project root) or `.claude/CLAUDE.md` (user-level)

Rules are freeform markdown. Organize under headings:
```markdown
## General Rules
- Rule about scope...

## Code Investigation
- Rule about verification...

## Testing
- Rule about test quality...
```

Keep rules:
- Concise and actionable
- Specific to observed friction (not generic advice)
- Testable ("only modify files explicitly requested" > "be careful with files")
