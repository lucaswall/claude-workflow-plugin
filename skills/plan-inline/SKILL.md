---
name: plan-inline
description: Create TDD implementation plans from direct feature requests. Use when user provides a task description like "add X feature", "create Y function", or "implement Z". Creates Linear issues in Todo state. Faster than plan-backlog for ad-hoc requests that don't need backlog tracking.
argument-hint: <task description>
allowed-tools: Read, Edit, Write, Glob, Grep, Task, Bash, mcp__linear__list_teams, mcp__linear__list_issues, mcp__linear__get_issue, mcp__linear__create_issue, mcp__linear__update_issue, mcp__linear__list_issue_labels, mcp__linear__list_issue_statuses
disable-model-invocation: true
---

Create a TDD implementation plan directly from inline instructions in $ARGUMENTS. Creates Linear issues in Todo state.

## Git Pre-flight Check

**Before doing anything else**, verify git state:

1. Check current branch: `git branch --show-current`
2. If NOT on `main` or `master`:
   - **STOP** with message: "Not on main branch. Please switch to main before planning: `git checkout main`"
3. Check for uncommitted changes: `git status --porcelain`
4. If there are uncommitted changes:
   - **STOP** with message: "Main branch has uncommitted changes. Please commit or stash them first."
5. Check if branch is up-to-date with remote: `git fetch origin && git status -uno`
6. If behind remote:
   - **STOP** with message: "Main branch is behind remote. Please pull latest: `git pull origin main`"

Only proceed to PLANS.md check if git state is clean.

## Purpose

- Convert inline task descriptions into actionable TDD implementation plans
- Create Linear issues in Todo state for each task (bypasses Backlog)
- Explore codebase to understand existing patterns and find relevant files
- Use MCPs to gather additional context (deployments, issue tracking)
- Generate detailed, implementable plans with full file paths and Linear issue links

## When to Use

Use `plan-inline` instead of `plan-backlog` when:
- The user provides a clear feature request or task description directly
- The task doesn't need to go through Linear Backlog first
- Quick planning without backlog management overhead

Use `plan-backlog` instead when:
- Working from existing backlog items
- Managing multiple items that should be tracked

## Pre-flight Check

**Before doing anything**, read PLANS.md and check for incomplete work:
- If PLANS.md has content but NO "Status: COMPLETE" at the end → **STOP**
- Tell the user: "PLANS.md has incomplete work. Please review and clear it before planning new items."
- Do not proceed.

If PLANS.md is empty or has "Status: COMPLETE" → proceed with planning.

**Verify Linear MCP:** Call `mcp__linear__list_teams`. If unavailable, **STOP** and tell the user: "Linear MCP is not connected. Run `/mcp` to reconnect, then re-run this skill."

## Discovering Team Context

Read CLAUDE.md to find the LINEAR INTEGRATION section. Look for:
- **Team name** (e.g., "Team: 'Food Scanner'" or "Team: 'ProjectName'")
- **Issue prefix** (e.g., "Prefix: PROJ-xxx")
- **State workflow** (e.g., "States: Backlog → Todo → In Progress → Review → Done")
- **Project-specific URLs** (Linear workspace URL, deployment URLs, etc.)

If CLAUDE.md doesn't have a LINEAR INTEGRATION section, call `mcp__linear__list_teams` to discover the team name dynamically.

Store the discovered team name in a variable for use throughout the skill.

## Arguments

$ARGUMENTS should contain the task description with context:
- What to implement or change
- Expected behavior
- Any constraints or requirements
- Related files if known

Example arguments:
- `Add a function to calculate nutritional score from scanned food data`
- `Create a new route /api/scan that processes food barcode images`
- `Update food detail page to show allergen warnings`

## Context Gathering

**IMPORTANT: Do NOT hardcode MCP names or folder paths.** Always read CLAUDE.md to discover:

1. **Available MCP servers** - Look for the "MCP SERVERS" section to find:
   - Railway MCP for deployment context (`get-logs`, `list-deployments`, `list-services`, `list-variables`)
   - Linear MCP for issue tracking (`list_issues`, `get_issue`, `create_issue`, etc.)

2. **Project structure** - Look for "STRUCTURE" section to understand:
   - Source code organization
   - Test file locations
   - Where to add new files

3. **Folder structure** - Look for "FOLDER STRUCTURE" section to understand:
   - Where components are stored
   - Naming conventions for files and folders

## Workflow

0. **Git pre-flight check** - Ensure on clean main branch (see Git Pre-flight Check section)
1. **Read PLANS.md** - Pre-flight check
2. **Read CLAUDE.md** - Understand TDD workflow, agents, project rules, available MCPs, discover team name
3. **Parse $ARGUMENTS** - Understand what needs to be implemented
4. **Explore codebase** - Use Glob/Grep/Task to find relevant files and understand patterns
5. **Gather MCP context** - If the task relates to:
   - Deployment → Check service status, recent logs
   - Existing issues → Check Linear for related issues or context
6. **Generate plan** - Create TDD tasks with test-first approach
7. **Write PLANS.md** - Overwrite with new plan
8. **Create Linear issues** - Create issues in Todo state for each task

## Codebase Exploration Guidelines

**When to explore:**
- Always explore to find existing patterns before creating new code
- Find related tests to understand testing conventions
- Locate where similar functionality already exists

**How to explore:**
- Use Glob for finding files by pattern: `src/**/*.ts`, `**/*.test.ts`
- Use Grep for finding code: function names, type definitions, error messages
- Use Task with `subagent_type=Explore` for broader questions about the codebase

**What to discover:**
- Existing functions that could be reused or extended
- Test file conventions and patterns
- Type definitions to reuse
- Similar implementations to follow as templates

## PLANS.md Structure

```markdown
# Implementation Plan

**Created:** YYYY-MM-DD
**Source:** Inline request: [Summary of $ARGUMENTS]
**Linear Issues:** [PROJ-123](https://linear.app/...), [PROJ-124](https://linear.app/...)

## Context Gathered

### Codebase Analysis
- **Related files:** [files found through exploration]
- **Existing patterns:** [patterns to follow]
- **Test conventions:** [how tests are structured in this area]

### MCP Context (if applicable)
- **MCPs used:** [which MCPs were consulted]
- **Findings:** [relevant information discovered]

## Original Plan

### Task 1: [Name]
**Linear Issue:** [PROJ-123](https://linear.app/...)

1. Write test in [file].test.ts for [function/scenario]
2. Run verifier (expect fail)
3. Implement [function] in [file].ts
4. Run verifier (expect pass)

### Task 2: [Name]
**Linear Issue:** [PROJ-124](https://linear.app/...)

1. Write test...
2. Run verifier...
3. Implement...
4. Run verifier...

## Post-Implementation Checklist
1. Run `bug-hunter` agent - Review changes for bugs
2. Run `verifier` agent - Verify all tests pass and zero warnings

---

## Plan Summary

**Objective:** [One sentence describing what this plan accomplishes]

**Request:** [Brief paraphrase of the original $ARGUMENTS]

**Linear Issues:** [PROJ-123, PROJ-124, ...]

**Approach:** [2-3 sentences describing the implementation strategy at a high level]

**Scope:**
- Tasks: [count]
- Files affected: [estimated count]
- New tests: [yes/no]

**Key Decisions:**
- [Important architectural or design decision 1]
- [Important decision 2, if any]

**Risks/Considerations:**
- [Any risks or things to watch out for]
```

## Linear Issue Creation

After writing PLANS.md, create a Linear issue for each task:

1. Use `mcp__linear__create_issue` with:
   - `team`: [Discovered team name from CLAUDE.md or `mcp__linear__list_teams`]
   - `title`: Task name
   - `description`: Task details from PLANS.md
   - `state`: "Todo"
   - `labels`: Infer from task type (Feature, Improvement, Bug)

2. Update PLANS.md to add `**Linear Issue:** [PROJ-N](url)` to each task

## Task Writing Guidelines

Each task must be:
- **Self-contained** - Full file paths, clear descriptions
- **TDD-compliant** - Test before implementation
- **Specific** - What to test, what to implement
- **Ordered** - Dependencies resolved by task order
- **Context-aware** - Reference patterns and files discovered during exploration

Good task example:
```markdown
### Task 1: Add calculateNutritionalScore function
1. Write test in src/utils/nutrition.test.ts for calculateNutritionalScore
   - Test valid food data returns correct score
   - Test missing nutrients returns partial score
   - Test empty input returns zero score
   - Follow existing utility function patterns
2. Run verifier (expect fail)
3. Implement calculateNutritionalScore in src/utils/nutrition.ts
   - Use Nutri-Score algorithm
   - Follow existing function signature patterns
4. Run verifier (expect pass)
```

Bad task example:
```markdown
### Task 1: Add nutrition scoring
1. Add function
2. Test it
```

## MCP Usage Guidelines

Discover available MCPs from CLAUDE.md's "MCP SERVERS" section. Common patterns:

**Deployment MCPs (Railway)** - Use when task involves:
- Deployment configuration
- Environment variables
- Service logs for debugging context

**Issue Tracking MCPs (Linear)** - Use when task involves:
- Checking existing issues for context
- Understanding related work
- Finding duplicate or related feature requests

If CLAUDE.md doesn't list MCPs, skip MCP context gathering.

## Error Handling

| Situation | Action |
|-----------|--------|
| PLANS.md has incomplete work | Stop and tell user to review/clear PLANS.md first |
| $ARGUMENTS is empty or unclear | Ask user to provide a clearer task description |
| CLAUDE.md doesn't exist | Continue without project-specific rules, use general TDD practices |
| Codebase exploration times out | Continue with partial context, note limitation in plan |
| MCP not available | Skip MCP context gathering, note in plan what was skipped |
| Task too vague to plan | Ask user for specific requirements before proceeding |

## Rules

- **Refuse to proceed if PLANS.md has incomplete work**
- **Explore codebase before planning** - Find patterns to follow
- **Use MCPs when relevant** - Gather context from external systems (discover from CLAUDE.md)
- Every task must follow TDD (test first, then implement)
- No manual verification steps - use agents only
- Tasks must be implementable without additional context
- Always include post-implementation checklist
- Create Linear issues in Todo state (bypasses Backlog)
- Include Linear issue links in PLANS.md tasks
- **Flag migration-relevant tasks** — If a task changes DB schema, renames columns, changes identity models, renames env vars, or changes session/token formats, add a note in the task: "**Migration note:** [what production data is affected]". The implementer will log this in `MIGRATIONS.md`.
- **Plans describe WHAT and WHY, not HOW at the code level.** Include: file paths, function names, behavioral specs, test assertions, patterns to follow (reference existing files by path), state transitions. Do NOT include: implementation code blocks, ready-to-paste TypeScript/TSX, full function bodies. The implementer (plan-implement workers) writes all code — your job is architecture and specification. Exception: short one-liners for surgical changes (e.g., "add `if (!session.x)` check after the existing `!session.y` check") are fine.

## CRITICAL: Scope Boundaries

**This skill creates plans. It does NOT implement them.**

1. **NEVER ask to "exit plan mode"** - This skill doesn't use Claude Code's plan mode feature
2. **NEVER implement code** - Your job ends when PLANS.md is written
3. **NEVER ask ambiguous questions** like "should I proceed?" or "ready to continue?"
4. **NEVER start implementing** after writing the plan, even if user says "yes" to something

## Termination

When you finish writing PLANS.md (and creating Linear issues), output the plan summary followed by the completion message:

```
Plan created in PLANS.md
Linear issues created in Todo: PROJ-123, PROJ-124, ...

## Plan Summary

**Objective:** [Copy from PLANS.md summary]

**Request:** [Copy from PLANS.md summary]

**Linear Issues:** [Copy from PLANS.md summary]

**Approach:** [Copy from PLANS.md summary]

**Scope:**
- Tasks: [count]
- Files affected: [estimated count]
- New tests: [yes/no]

**Key Decisions:**
- [List from PLANS.md summary]

**Risks/Considerations:**
- [List from PLANS.md summary]

---

Create a feature branch and commit the plan.
```

**Then execute git workflow:**

1. Create a feature branch with proper naming:
   - Use `feat/` prefix for new features
   - Use `fix/` prefix for bug fixes
   - Use `refactor/` prefix for refactoring
   - Branch name should be kebab-case, derived from the plan objective
   - Example: `feat/nutritional-score-calculator`, `refactor/extract-common-scanner-utils`

2. Stage, commit (no `Co-Authored-By` tags), and push:
```bash
git checkout -b <type>/<task-description> && git add PLANS.md && git commit -m "plan: <task-description>" && git push -u origin <type>/<task-description>
```

Do not ask follow-up questions. Do not offer to implement. Output the summary and stop.
