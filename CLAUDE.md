# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

`lw-workflow` is a Claude Code plugin providing a complete TDD development workflow. It consists of 14 skills and 3 agents orchestrating planning, implementation, review, auditing, and release of TypeScript projects. There is no build system — the plugin is pure markdown.

**Installation:** `claude --add-dir /path/to/claude-workflow-plugin`
**Skills invoked as:** `/lw-workflow:<skill-name>` (e.g., `/lw-workflow:plan-inline`)

## Architecture

### Plugin Structure

- `.claude-plugin/plugin.json` — Plugin manifest (name, version, author)
- `agents/*.md` — Agent definitions (model, permissions, tools, instructions)
- `skills/<name>/instructions.md` — Skill instructions (the prompt)
- `skills/<name>/metadata.yml` — Skill metadata (triggers, invocation rules)
- `skills/<name>/references/*.md` — Reference documents loaded by skills

### Skill Categories

| Category | Skills | Purpose |
|----------|--------|---------|
| **Planning** | plan-inline, plan-fix, plan-backlog, add-to-backlog, backlog-refine | Create TDD plans, manage Linear issues |
| **Implementation** | plan-implement, plan-review-implementation | Execute plans with agent teams, QA review |
| **Auditing** | code-audit, frontend-review, deep-review | Security/quality/UX audits with multi-reviewer teams |
| **Research** | investigate, pull-from-roadmap | Read-only investigation, feature research |
| **Release** | push-to-production | DB backup, migration, deploy |
| **Meta** | tools-improve | Best practices for modifying this plugin |

### Agents

| Agent | Model | Purpose |
|-------|-------|---------|
| **verifier** | Haiku | Runs tests/lint/build (3 modes: TDD filtered, Full, E2E) |
| **pr-creator** | Sonnet | Creates PRs with Linear issue links |
| **bug-hunter** | Sonnet | OWASP-based code review of git changes |

### Key Patterns

**CLAUDE.md Integration:** Skills discover target project context from sections in that project's CLAUDE.md: LINEAR INTEGRATION, STRUCTURE, ENVIRONMENTS, COMMANDS, MCP SERVERS. Fallback to dynamic MCP discovery if sections are missing.

**Linear State Machine:** Backlog → Todo → In Progress → Review → Merge → Done → Released. Each skill moves issues through specific transitions.

**Agent Team Orchestration:** Team skills (plan-implement, plan-review-implementation, code-audit, frontend-review) use `TeamCreate` to spawn parallel workers. File-ownership partitioning ensures no file is assigned to multiple workers. All team skills include single-agent fallback if `TeamCreate` fails.

**TDD Loop:** Planning skills enforce test-first: write tests → verifier (expect fail) → implement → verifier (expect pass) → bug-hunter review.

**PLANS.md:** The shared document between planning and implementation skills. Contains context gathered, tasks with test/implementation steps, Linear issue links, and iteration tracking.

## Modifying This Plugin

**Always load `/lw-workflow:tools-improve` first** before creating or modifying skills/agents. It provides templates and reference docs for:
- `skills-reference.md` — Skill metadata options, string substitutions (`$ARGUMENTS`, `$0`-`$N`)
- `subagents-reference.md` — Agent definitions, model/permission options
- `agent-teams-reference.md` — Team coordination patterns, task list, messaging
- `claude-md-reference.md` — CLAUDE.md best practices for target projects

### Skill Conventions

- Each skill is a directory under `skills/` with `instructions.md` and `metadata.yml`
- Reference files go in `skills/<name>/references/` and are loaded via `$ref:` in metadata
- Skills use progressive disclosure: metadata (layer 1) → instructions (layer 2) → references (layer 3)
- MCP tool names are never hardcoded — discovered from CLAUDE.md or dynamically

### Agent Conventions

- Each agent is a single `.md` file under `agents/`
- Permission levels: `dontAsk` (auto-approved), `bypassPermissions` (full access)
- Workers use Haiku for speed (verifier) or Sonnet for reasoning (pr-creator, bug-hunter)
- Agent team workers are capped at 4; lead handles all MCP and Linear operations
