# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

`lw-workflow` is a Claude Code plugin providing a complete TDD development workflow. It consists of 15 skills and 3 agents orchestrating planning, implementation, review, auditing, and release of TypeScript projects. There is no build system — the plugin is pure markdown.

**Installation:** `claude --add-dir /path/to/claude-workflow-plugin`
**Skills invoked as:** `/lw-workflow:<skill-name>` (e.g., `/lw-workflow:plan-inline`)

**Self-use:** Skills cannot be invoked from within this plugin project directly. To use a skill here (e.g., `tools-improve`), read its `SKILL.md` and references manually.

## Architecture

### Plugin Structure

- `.claude-plugin/plugin.json` — Plugin manifest (name, version, author)
- `agents/*.md` — Agent definitions (YAML frontmatter + markdown instructions)
- `skills/<name>/SKILL.md` — Skill definition (YAML frontmatter for metadata + markdown body for instructions)
- `skills/<name>/references/*.md` — Supporting documents loaded by skills as needed

### Skill Categories

| Category | Skills | Purpose |
|----------|--------|---------|
| **Planning** | plan-inline, plan-fix, plan-backlog, add-to-backlog, backlog-refine | Create TDD plans, manage Linear issues |
| **Implementation** | plan-implement, plan-review-implementation | Execute plans with agent teams, QA review |
| **Auditing** | code-audit, frontend-review, deep-review | Security/quality/UX audits with multi-reviewer teams |
| **Research** | investigate, pull-from-roadmap | Read-only investigation, feature research |
| **Release** | push-to-production | DB backup, migration, deploy |
| **Meta** | tools-improve, tools-migrate | Best practices for modifying this plugin, bidirectional skill/agent migration |

### Agents

| Agent | Model | Permission Mode | Purpose |
|-------|-------|-----------------|---------|
| **verifier** | Haiku | dontAsk | Runs tests/lint/build (3 modes: TDD filtered, Full, E2E) |
| **pr-creator** | Sonnet | bypassPermissions | Creates PRs with Linear issue links |
| **bug-hunter** | Sonnet | dontAsk | OWASP-based code review of git changes |

### Key Patterns

**CLAUDE.md Integration:** Skills discover target project context from sections in that project's CLAUDE.md: LINEAR INTEGRATION, STRUCTURE, ENVIRONMENTS, COMMANDS, MCP SERVERS. Fallback to dynamic MCP discovery if sections are missing.

**Linear State Machine:** Backlog → Todo → In Progress → Review → Merge → Done → Released. Each skill moves issues through specific transitions.

**Agent Team Orchestration:** Team skills (plan-implement, plan-review-implementation, code-audit, frontend-review) use `TeamCreate` to spawn parallel workers. Implementation workers (plan-implement) get isolated git worktrees with domain-based task assignment; review workers share the main directory. All team skills include single-agent fallback if `TeamCreate` fails.

**TDD Loop:** Planning skills enforce test-first: write tests → verifier (expect fail) → implement → verifier (expect pass) → bug-hunter review.

**PLANS.md:** The shared document between planning and implementation skills. Contains context gathered, tasks with test/implementation steps, Linear issue links, and iteration tracking.

## Modifying This Plugin

**IMPORTANT: Read `skills/tools-improve/SKILL.md` and its `references/` directory first** before creating or modifying skills/agents. It provides templates and reference docs for:
- `skills-reference.md` — Skill frontmatter fields, string substitutions (`$ARGUMENTS`, `$0`-`$N`), progressive disclosure
- `subagents-reference.md` — Agent definitions, permission modes, hooks, MCP access
- `agent-teams-reference.md` — Team coordination patterns, task list, messaging
- `claude-md-reference.md` — CLAUDE.md best practices for target projects

### Skill Conventions

- Each skill is a directory under `skills/` with a single `SKILL.md` file (YAML frontmatter for metadata, markdown body for instructions)
- Reference files go in `skills/<name>/references/` and are read by the skill instructions as needed
- Skills use progressive disclosure: metadata (~100 tokens, always loaded) → instructions (<5k tokens, on trigger) → references (unlimited, on demand)
- All skills set `disable-model-invocation: true` — user must invoke explicitly
- Linear MCP tool names (e.g., `mcp__linear__*`) are listed in `allowed-tools` frontmatter for permissions, but skill instructions discover other MCPs dynamically from the target project's CLAUDE.md

### Agent Conventions

- Each agent is a single `.md` file under `agents/` (YAML frontmatter + markdown instructions)
- Permission modes: `dontAsk` (auto-deny prompts, use for read-only agents), `bypassPermissions` (skip all checks)
- Workers use Haiku for speed (verifier) or Sonnet for reasoning (pr-creator, bug-hunter)
- Agent team workers are capped at 4; lead handles all MCP and Linear operations
