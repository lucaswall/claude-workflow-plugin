# Subagents Quick Reference

## Built-in Subagents

| Agent | Model | Tools | Use For |
|-------|-------|-------|---------|
| **Explore** | Haiku | Read-only (denied Write/Edit) | Fast codebase exploration, file discovery |
| **Plan** | Inherit | Read-only (denied Write/Edit) | Research during plan mode |
| **general-purpose** | Inherit | All | Complex multi-step tasks |
| **Bash** | Inherit | Bash | Terminal commands in separate context |
| **statusline-setup** | Sonnet | - | `/statusline` configuration |
| **Claude Code Guide** | Haiku | - | Questions about Claude Code features |

## Managing Subagents

**`/agents` command** — Interactive UI for viewing, creating, editing, and deleting subagents. Recommended for management. Subagents created via `/agents` are available immediately without restart.

**Manual files** — Create `.md` files directly. Requires session restart or `/agents` to load.

## Key Constraints

**Max 3-4 custom subagents** - Too many reduces productivity and confuses delegation.

**Subagents cannot spawn other subagents.** If your workflow requires nested delegation, use skills or chain subagents from the main conversation.

**Restrict spawnable agents** — Use `Task(type1, type2)` in the `tools` field to allowlist which agent types can be spawned (only applies to agents running as main thread with `claude --agent`).

## Where Subagents Live

| Location | Scope | Priority |
|----------|-------|----------|
| `--agents` CLI flag (JSON) | Current session only | 1 (highest) |
| `.claude/agents/` | Current project | 2 |
| `~/.claude/agents/` | All your projects | 3 |
| Plugin's `agents/` directory | Where plugin is enabled | 4 (lowest) |

## Permission Modes

| Mode | Behavior | Use When |
|------|----------|----------|
| `default` | Standard prompts | General purpose |
| `acceptEdits` | Auto-accept Write/Edit | Trusted code writers |
| `delegate` | Coordination-only (team management tools only) | Agent team leads |
| `dontAsk` | Auto-deny prompts (allowed tools still work) | **Read-only agents** |
| `bypassPermissions` | Skip ALL checks (dangerous) | Rarely - high trust only |
| `plan` | Read-only exploration | Planning/research |

**Tip:** Use `dontAsk` for read-only agents (reviewers, explorers, auditors). It auto-denies write operations while allowing the tools in the `tools` list.

**Note:** If the parent uses `bypassPermissions`, this takes precedence and cannot be overridden.

## MCP Access

**Subagents inherit MCP tools by default** when tools are not restricted. Configure per-subagent MCP with the `mcpServers` field (name reference to an already-configured server, or inline definition).

**Background subagents do NOT have MCP access.** Only foreground subagents can use MCP tools.

**Agent team teammates** — The docs state teammates load MCP servers at spawn, but practical experience shows MCP tools may not work reliably for teammates. Keep MCP operations on the lead.

## Persistent Memory

The `memory` field gives a subagent a persistent directory that survives across conversations:

| Scope | Location | Use When |
|-------|----------|----------|
| `user` | `~/.claude/agent-memory/<name>/` | Knowledge across all projects (recommended default) |
| `project` | `.claude/agent-memory/<name>/` | Project-specific, shareable via VCS |
| `local` | `.claude/agent-memory-local/<name>/` | Project-specific, NOT in VCS |

When enabled:
- System prompt includes read/write instructions for the memory directory
- First 200 lines of `MEMORY.md` are auto-loaded into the system prompt
- Read, Write, Edit tools automatically enabled

## Hook Events

| Event | Matcher | When |
|-------|---------|------|
| `PreToolUse` | Tool name | Before tool executes |
| `PostToolUse` | Tool name | After tool executes |
| `Stop` | (none) | Subagent finishes (converted to `SubagentStop` at runtime) |
| `SubagentStart` | Agent name | Subagent begins (settings.json only) |
| `SubagentStop` | (none) | Any subagent completes (settings.json only) |
| `TeammateIdle` | (none) | Teammate goes idle after a turn |
| `TaskCompleted` | (none) | A task is marked completed |

### Hook Input (stdin JSON)
```json
{
  "session_id": "abc123",
  "tool_name": "Bash",
  "tool_input": { "command": "npm test" }
}
```

### Hook Exit Codes
| Code | Behavior |
|------|----------|
| 0 | Continue normally |
| 1 | Error (shows message) |
| 2 | Block the operation |

## Foreground vs Background

**Foreground** (default):
- Blocks main conversation
- Permission prompts pass through
- Can ask clarifying questions
- MCP tools available

**Background**:
- Runs concurrently
- Permissions pre-approved at launch, auto-denies anything not pre-approved
- Cannot ask questions (tool call fails but subagent continues)
- **No MCP tools**

Trigger background: ask "run in background" or press **Ctrl+B**

Disable background: `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS=1`

## Resume Subagents

Subagents can be resumed to continue previous work with full context:
```
Continue that code review and analyze authorization
```

Transcripts: `~/.claude/projects/{project}/{sessionId}/subagents/agent-{agentId}.jsonl`

Transcripts persist independently of main conversation compaction and survive session restarts. Cleaned up based on `cleanupPeriodDays` setting (default: 30 days).

## Preload Skills into Subagents

The `skills` field **injects full skill content** into the subagent's context at startup (not just made available for invocation). Subagents don't inherit skills from the parent conversation.

```yaml
skills:
  - api-conventions
  - error-handling-patterns
```

This is the inverse of `context: fork` in a skill. With `skills` in a subagent, the subagent controls the system prompt and loads skill content. With `context: fork` in a skill, the skill content becomes the task for the agent.

## Auto-Compaction

Subagents auto-compact at ~95% capacity (same as main conversation).

To trigger earlier: `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=50` (percentage)

## Complete Example

```yaml
---
name: security-auditor
description: Security audit specialist. Use proactively after modifying auth code.
tools: Read, Grep, Glob, Bash
disallowedTools: Write, Edit
model: sonnet
permissionMode: dontAsk
memory: user
skills:
  - security-patterns
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./scripts/validate-safe-commands.sh"
  Stop:
    - hooks:
        - type: command
          command: "./scripts/cleanup.sh"
---

You are a security auditor. When invoked:
1. Run git diff to see recent changes
2. Focus on authentication and authorization code
3. Check for OWASP Top 10 vulnerabilities
4. Report findings by severity: Critical, High, Medium, Low

Never modify files. Only analyze and report.
```

## Disable Subagents

In settings.json:
```json
{
  "permissions": {
    "deny": ["Task(Explore)", "Task(my-custom-agent)"]
  }
}
```

Or via CLI:
```bash
claude --disallowedTools "Task(Explore)"
```
