---
name: tools-improve
description: "REQUIRED: Load this skill BEFORE creating, modifying, or reviewing any Claude Code skill, subagent, or CLAUDE.md file. Contains critical best practices for .claude/skills/, .claude/agents/, and CLAUDE.md. Use when: creating agents, creating skills, editing SKILL.md files, editing agent .md files, reviewing skill/agent code, reviewing or editing CLAUDE.md, adding instructions to CLAUDE.md, or any work involving Claude Code extensibility."
argument-hint: <skill or agent name>
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# Tools Improve - Agent, Skill & CLAUDE.md Assistant

You create and optimize Claude Code subagents, skills, and CLAUDE.md files.

## Workflow

**For skills/agents:**
1. **Clarify requirements** - What does user want to create/modify?
2. **Decide type** - Skill vs subagent (see Decision section below)
3. **Create/edit file** - Use appropriate template
4. **Update CLAUDE.md** - Add new skill/agent to the SKILLS or SUBAGENTS table
5. **Verify** - Confirm description triggers correctly for auto-discovery

**For CLAUDE.md:** Read [references/claude-md-reference.md](references/claude-md-reference.md) for the full review checklist and inclusion/exclusion criteria (sourced from Anthropic official docs).

## Reference Docs

For detailed information beyond this file:
- [references/claude-md-reference.md](references/claude-md-reference.md) - CLAUDE.md include/exclude criteria, review checklist, modular organization
- [skills-reference.md](skills-reference.md) - Invocation control, context budget, hooks, progressive disclosure, testing, troubleshooting
- [subagents-reference.md](subagents-reference.md) - Built-in agents, permission modes, hook events, MCP access, memory, resume
- [agent-teams-reference.md](agent-teams-reference.md) - Team orchestration, task coordination, display modes, troubleshooting

## Decision: Skill vs Subagent

**Use SUBAGENT when:**
- Task produces high-volume output (tests, logs, exploration)
- Need parallel execution (concurrent research tasks)
- Need strict tool restrictions with validation hooks
- Work is self-contained and returns a summary

**Use SKILL when:**
- Reusable instructions/knowledge for main conversation
- Side effects user must control → `disable-model-invocation: true`
- Background knowledge Claude should apply → `user-invocable: false`
- Quick reference or style guides

**Default to skill** - simpler, runs in main context.

**When a skill needs parallel workers**, it can orchestrate an **agent team** internally. See "Agent Teams in Skills" under Best Practices and [agent-teams-reference.md](agent-teams-reference.md) for the full guide.

## Templates

### Skill Template
Create `.claude/skills/<name>/SKILL.md`:
```yaml
---
name: my-skill
description: What it does. Use when [triggers]. Helps with [use cases].
disable-model-invocation: true  # Add for workflows with side effects
---

Instructions Claude follows when skill is invoked.
```

**Supporting files** go in the skill directory:
```
my-skill/
├── SKILL.md              # Main instructions (required)
├── references/           # Detailed docs loaded when needed
│   └── checklist.md
└── scripts/              # Executable scripts
    └── helper.sh
```

### Subagent Template
Create `.claude/agents/<name>.md`:
```yaml
---
name: my-agent
description: What it does. Use proactively when [triggers].
tools: Read, Glob, Grep, Bash
model: sonnet
permissionMode: dontAsk  # Add for read-only agents (auto-deny writes)
---

You are a specialist in [domain]. When invoked:
1. [First action]
2. [Second action]
3. Return summary of findings/changes.
```

## Frontmatter Reference

### Skill Fields (SKILL.md)
| Field | Description |
|-------|-------------|
| `name` | Slash command name (defaults to directory). Lowercase, numbers, hyphens only, max 64 chars |
| `description` | **Critical** - triggers auto-discovery. Falls back to first paragraph if omitted |
| `argument-hint` | Autocomplete hint: `[issue-number]` |
| `disable-model-invocation` | `true` = only user can invoke |
| `user-invocable` | `false` = hidden from `/` menu |
| `allowed-tools` | Tools without permission prompts |
| `model` | Model override: sonnet, opus, haiku |
| `context` | `fork` = run in isolated subagent |
| `agent` | Subagent type when forked |
| `hooks` | Lifecycle hooks (PreToolUse, PostToolUse) |

### Subagent Fields (.md)
| Field | Description |
|-------|-------------|
| `name` | Unique identifier (lowercase, hyphens). Required |
| `description` | **Critical** - when Claude delegates. Required |
| `tools` | Allowlist (inherits all including MCP if omitted). Use `Task(type1, type2)` to restrict spawnable agents |
| `disallowedTools` | Denylist from inherited tools |
| `model` | sonnet, opus, haiku, inherit (default: inherit) |
| `permissionMode` | default, acceptEdits, delegate, dontAsk, bypassPermissions, plan |
| `maxTurns` | Max agentic turns before stopping |
| `skills` | Preload full skill content at startup |
| `mcpServers` | MCP servers available (name reference or inline config) |
| `memory` | Persistent memory scope: `user`, `project`, or `local` |
| `hooks` | PreToolUse, PostToolUse, Stop |

### String Substitutions (Skills)
| Variable | Description |
|----------|-------------|
| `$ARGUMENTS` | All arguments passed |
| `$ARGUMENTS[N]` or `$N` | Positional argument (0-indexed) |
| `${CLAUDE_SESSION_ID}` | Current session ID |
| `!{backtick}command{backtick}` | Dynamic command output (runs before Claude sees content). Syntax: exclamation + backtick-wrapped shell command |

## Patterns

### Read-Only Reviewer (Subagent)
```yaml
---
name: code-reviewer
description: Reviews code quality. Use proactively after writing code.
tools: Read, Grep, Glob, Bash
model: sonnet
---

Review code for quality, security, and best practices.
Run git diff to see changes. Provide feedback by priority.
```

### Side-Effect Action (Skill)
```yaml
---
name: deploy
description: Deploy to production
disable-model-invocation: true
context: fork
allowed-tools: Bash, Read
---

Deploy $ARGUMENTS:
1. Run tests
2. Build
3. Push to target
```

### Background Knowledge (Skill)
```yaml
---
name: legacy-context
description: Context about legacy system. Use when working with payment code.
user-invocable: false
---

Legacy payment system uses SOAP API, XML config, stored procedures.
Key files: src/payments/legacy-adapter.ts
```

### Team-Orchestrating Skill
```yaml
---
name: parallel-review
description: Parallel code review with specialized reviewers
allowed-tools: Read, Glob, Grep, Task, Bash, TeamCreate, TeamDelete,
  SendMessage, TaskCreate, TaskUpdate, TaskList, TaskGet
disable-model-invocation: true
---

1. TeamCreate → TaskCreate (one per reviewer) → spawn teammates
2. Wait for findings messages (auto-delivered)
3. Merge, deduplicate, act on results
4. Shutdown teammates → TeamDelete
Fallback: if TeamCreate fails, use sequential Task subagents
```

### Hook Validation (Subagent)
```yaml
---
name: db-reader
description: Execute read-only SQL queries
tools: Bash
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./scripts/validate-readonly.sh"
---
```

Hook script (exit 2 to block):
```bash
#!/bin/bash
INPUT=$(cat)
CMD=$(echo "$INPUT" | jq -r '.tool_input.command // empty')
if echo "$CMD" | grep -iE '\b(INSERT|UPDATE|DELETE|DROP)\b' >/dev/null; then
  echo "Blocked: Only SELECT allowed" >&2
  exit 2
fi
exit 0
```

## Best Practices

### Descriptions Are Critical

Claude uses descriptions for auto-discovery. Follow the **[what] + [when] + [features]** formula:

**Poor:** `"Helps with code"`
**Good:** `"Explains code using diagrams and analogies. Use when describing how code works, teaching about codebases, or answering 'how does this work?' questions."`

Test: ask Claude "When should this skill be used?" — if the answer doesn't match your intent, refine the description.

### Progressive Disclosure

Skills load in 3 stages to optimize context:
1. **Metadata** (~100 tokens/skill) — `name` + `description` always in context
2. **Instructions** (<5k tokens) — SKILL.md body loaded when triggered
3. **Resources** (unlimited) — Supporting files loaded as needed via references

Design skills with this in mind: keep SKILL.md focused; put large docs in supporting files.

### Skill Architecture

**Keep SKILL.md under 500 lines** - Use supporting files in the skill's own directory for details.

**Skills are self-contained** - Each skill directory is independent. There is NO shared directory pattern across skills. Supporting files go in `<skill>/references/` or similar subdirectories within that skill.

**Extended thinking** — Include the word "ultrathink" in skill content to enable extended thinking.

**Reducing duplication across skills:**
1. **Put in CLAUDE.md** - Content loaded into all contexts (best for project-wide conventions)
2. **Background knowledge skill** - Use `user-invocable: false` for shared knowledge Claude auto-loads
3. **Accept duplication** - Self-contained skills are more maintainable than fragile dependencies

### Invocation Control

| Scenario | Frontmatter |
|----------|-------------|
| Side effects (deploy, commit, modify files) | `disable-model-invocation: true` |
| Background knowledge (not a command) | `user-invocable: false` |
| Safe for Claude to auto-invoke | (default - no flags) |

### Tool & Permission Restrictions

**Limit tool access** - Grant only what's needed via `tools` or `allowed-tools`.

**Permission modes for subagents:**
- `default` - Standard prompts
- `dontAsk` - Auto-deny prompts (use for read-only agents)
- `acceptEdits` - Auto-accept file edits
- `delegate` - Coordination-only (for agent team leads, restricts to team management tools)
- `bypassPermissions` - Skip all checks (dangerous)

### Model Selection

Match model to task complexity:
- `haiku` - Fast, cheap (exploration, simple validation, tests)
- `sonnet` - Balanced (code review, git operations, implementation)
- `opus` - Complex reasoning, bug detection, critical decisions

### Subagent Limits

**Max 3-4 custom subagents** - Too many reduces productivity and confuses delegation.

### Agent Teams in Skills

When a skill needs parallel workers (code review, parallel implementation, competing hypotheses), it can orchestrate an agent team. Requires `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`. See [agent-teams-reference.md](agent-teams-reference.md) for full guide.

**When to use teams inside a skill:**
- Workers need to communicate with each other (not just report back to lead)
- Work benefits from domain specialization (security vs reliability vs quality)
- Parallel implementation across non-overlapping file sets
- Higher token cost is justified by speed or depth

**Key rules for team-orchestrating skills:**
1. **Isolate workers that write files** — Implementation workers get git worktrees (`_workers/worker-N/`) with own branch; domain-based assignment allowed. Review-only workers (code-audit, frontend-review) share the main directory safely. **Fallback:** partition by file ownership if worktrees unavailable
2. **Lead handles external writes** — Teammates lack MCP access; lead does Linear/Railway/etc.
3. **Domain specialization** — Assign distinct domains, not "review the code" to everyone
4. **Structured reporting** — Define a findings format so lead can merge and deduplicate
5. **Always include fallback** — If `TeamCreate` fails, fall back to sequential subagents
6. **Cap at ~4 teammates** — Diminishing returns beyond that for most tasks
7. **Don't let workers hand-write generated files** — Reserve CLI generators for the lead

**Required `allowed-tools` for team skills:**
```
TeamCreate, TeamDelete, SendMessage, TaskCreate, TaskUpdate, TaskList, TaskGet, Task
```

**Skill lifecycle with teams:**
1. Pre-flight (verify dependencies)
2. `TeamCreate` → `TaskCreate` (one per work unit) → spawn teammates via `Task` with `team_name`
3. Assign tasks via `TaskUpdate`
4. Wait for teammate messages (auto-delivered, don't poll)
5. Merge/synthesize findings
6. Act on results (lead handles all external writes)
7. Shutdown teammates → `TeamDelete`

## File Locations

| Type | Project | User (all projects) |
|------|---------|---------------------|
| Skill | `.claude/skills/<name>/SKILL.md` | `~/.claude/skills/<name>/SKILL.md` |
| Subagent | `.claude/agents/<name>.md` | `~/.claude/agents/<name>.md` |

Priority: CLI flag > project > user > plugin

## Official Docs

For complete reference, see:
- Skills: https://code.claude.com/docs/en/skills
- Subagents: https://code.claude.com/docs/en/sub-agents
- Agent Teams: https://code.claude.com/docs/en/agent-teams

**Local references:** See [skills-reference.md](skills-reference.md), [subagents-reference.md](subagents-reference.md), and [agent-teams-reference.md](agent-teams-reference.md) for quick lookup tables.
