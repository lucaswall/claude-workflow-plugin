---
name: plan-backlog
description: Convert Linear Backlog issues into TDD implementation plans. Use when user says "plan PROJ-123", "plan all bugs", "work on backlog", or wants to implement issues from Linear. With no arguments, plans ALL Backlog issues. Moves planned issues to Todo state. Explores codebase for patterns and discovers available MCPs from CLAUDE.md.
argument-hint: [issue-selector] (optional) e.g., "PROJ-123", "all Bug issues" — omit to plan all Backlog items
allowed-tools: Read, Edit, Write, Glob, Grep, Task, Bash, mcp__linear__list_teams, mcp__linear__list_issues, mcp__linear__get_issue, mcp__linear__update_issue, mcp__linear__create_comment, mcp__linear__list_issue_labels, mcp__linear__list_issue_statuses
disable-model-invocation: true
---

# Plan Backlog Skill

Convert Linear Backlog issues into a structured TDD implementation plan written to `PLANS.md`.

## Overview

This skill takes one or more Linear issues from the Backlog state and produces a detailed, step-by-step TDD implementation plan. The plan is written to `PLANS.md` at the project root. After planning, the Linear issues are moved to the "Todo" state.

This skill creates plans. It does NOT implement them.

---

## Workflow

### Phase 1: Pre-Flight Checks

#### 1.1 Git Pre-Flight

Before doing anything, verify the git state:

```bash
git status
git branch --show-current
```

**Requirements:**
- Must be on `main` branch
- Working tree must be clean (no uncommitted changes)

If either check fails, STOP and report the issue to the user. Do not proceed.

#### 1.2 PLANS.md Pre-Flight

Check if `PLANS.md` exists at the project root:

```bash
ls -la PLANS.md
```

**Rules:**
- If `PLANS.md` does not exist: OK, proceed.
- If `PLANS.md` exists and its status is `COMPLETE`: OK, proceed (it will be overwritten).
- If `PLANS.md` exists and its status is NOT `COMPLETE`: STOP. Tell the user there is an active plan that must be completed or removed first.

To check status, read the file and look for EITHER:
- `**Status:** COMPLETE` in the header (line 3), OR
- `## Status: COMPLETE` anywhere in the file (appended by plan-review-implementation)

If either marker is found, the plan is complete.

#### 1.3 Verify Linear MCP

Call `mcp__linear__list_teams`. If unavailable, **STOP** and tell the user: "Linear MCP is not connected. Run `/mcp` to reconnect, then re-run this skill."

---

### Phase 2: Gather Context

#### 2.1 Query Linear for Backlog Issues

Use the Linear MCP to find the requested issues.

**First, discover the team name:**

Read CLAUDE.md and look for LINEAR INTEGRATION section. Extract the team name from patterns like:
- "Team: 'Food Scanner'"
- "Team: ProjectName"

If CLAUDE.md doesn't have a LINEAR INTEGRATION section, call `mcp__linear__list_teams` to discover the team name dynamically.

Store the discovered team name in a variable for use throughout the skill.

**If user specified a specific issue (e.g., "PROJ-123"):**

```
mcp__linear__get_issue(issueId: "PROJ-123")
```

Verify the issue exists and is in the "Backlog" state. If not in Backlog, warn the user but continue if they confirm.

**If user specified a filter (e.g., "all Bug issues", "the auth issue"):**

First, get the team's issue statuses and labels:

```
mcp__linear__list_issue_statuses(team: [discovered team name])
mcp__linear__list_issue_labels(team: [discovered team name])
```

Then query for Backlog issues:

```
mcp__linear__list_issues(team: [discovered team name], state: "Backlog", includeArchived: false)
```

Filter the results based on the user's criteria (label, title keywords, etc.).

**If user said "plan all", "work on backlog", or provided no arguments:**

```
mcp__linear__list_issues(team: [discovered team name], state: "Backlog", includeArchived: false)
```

Plan ALL returned Backlog issues. No confirmation needed — the triage phase (Phase 3) will filter out invalid ones.

#### 2.2 Read CLAUDE.md

Read the project's `CLAUDE.md` file to understand:
- Project architecture and conventions
- Available MCP servers and their capabilities
- Testing patterns and preferences
- Code style and structure guidelines

```
Read CLAUDE.md
```

This is critical for generating plans that align with the project's patterns.

#### 2.3 Explore the Codebase

Explore the codebase to understand existing patterns:

```bash
# Project structure
find . -type f -name "*.ts" -o -name "*.tsx" | head -50
find . -type f -name "*.test.*" | head -20

# Package dependencies
cat package.json

# Existing patterns
ls -la src/app/
ls -la src/components/
ls -la src/lib/
```

Use Glob and Grep to find:
- Existing components similar to what the issues require
- Test file patterns and conventions
- API route patterns
- Database/data patterns
- Shared utilities and hooks

#### 2.4 Gather MCP Context

Based on what you learned from CLAUDE.md, identify which MCP servers are available. Common ones for this project:

- **Linear MCP**: Issue tracking, status updates
- **Railway MCP**: Deployment and infrastructure context

Query relevant MCPs to gather context that will inform the plan. For example:
- Check Railway for existing services and environment variables
- Check Linear for related issues or dependencies

---

### Phase 3: Triage Issues

Before planning, assess whether each backlog issue is **valid and actionable** in the current project context. Issues from code audits may flag theoretical problems that don't apply.

#### 3.1 Validate Each Issue

For each candidate issue, read the referenced code and ask:

1. **Does the problem actually exist?** Read the file/line cited in the issue. Is the code actually there? Does it behave as the issue claims?
2. **Is it relevant to the project context?** Consider:
   - Project status (DEVELOPMENT = no legacy data, no backward compatibility)
   - Single-user vs multi-user implications
   - Client-side vs server-side distinctions
   - Whether the "fix" is already the correct behavior
3. **Is it a real risk or a theoretical concern?** A single-user app behind auth doesn't need the same defenses as a public API.
4. **Is it already addressed?** Check if another issue or existing code already handles this.

#### 3.2 Classify Issues

Place each issue in one of two categories:

| Category | Criteria | Action |
|----------|----------|--------|
| **Valid** | Problem is real, fix is actionable, applies to current context | Include in plan |
| **Invalid** | Problem doesn't exist, is theoretical, or "fix" would be wrong | Cancel the issue |

#### 3.3 Cancel Invalid Issues

For each invalid issue:

1. **Add a comment** explaining why the issue is being canceled:

```
mcp__linear__create_comment(issueId: "PROJ-xxx", body: "Canceled during triage: [reason]")
```

The reason should be specific, e.g.:
- "Project is in DEVELOPMENT status with no legacy data — plaintext token migration path is dead code."
- "console.error is the correct choice for client-side error boundaries — pino is server-only."
- "Single-user app behind auth — x-forwarded-for spoofing is not a real risk."

2. **Move to Canceled state.**

**CRITICAL: Linear MCP same-type state bug.** "Duplicate" and "Canceled" are both `type: canceled` in Linear. Passing `state: "Canceled"` by name silently no-ops if the issue is already in another canceled-type state. To reliably cancel issues, first fetch the team's statuses to get the Canceled state UUID:

```
mcp__linear__list_issue_statuses(team: [discovered team name])
```

Find the status with `name: "Canceled"` and use its `id` (UUID) in the update call:

```
mcp__linear__update_issue(id: "PROJ-xxx", state: "<canceled-state-uuid>")
```

**Always use the UUID, never the name**, for canceled-type state transitions.

#### 3.4 Report Triage Results

Before proceeding, present the triage results to the user:

```
## Triage Results

**Valid (will be planned):**
- PROJ-123: [title] — [brief reason it's valid]
- PROJ-456: [title] — [brief reason it's valid]

**Canceled:**

| Issue | Title | Reason |
|-------|-------|--------|
| PROJ-789 | [title] | [specific reason it's invalid] |
| PROJ-012 | [title] | [specific reason it's invalid] |
```

Document canceled issues in the plan's **Scope Boundaries → Out of Scope** section with the cancellation reason.

If ALL issues are invalid, STOP — inform the user that no issues need planning.

If valid issues remain, proceed to Phase 4.

---

### Phase 4: Generate the Plan

#### 4.1 Analyze Requirements

For each issue being planned:
1. Read the issue title, description, and any comments
2. Identify acceptance criteria (explicit or implied)
3. Identify dependencies on other issues or existing code
4. Determine the scope of changes needed
5. Identify which files will be created or modified

#### 4.2 Design the Implementation

For each issue:
1. Break down into small, testable tasks
2. Order tasks so each builds on the previous
3. Identify the TDD cycle for each task (test first, then implement)
4. Note any MCP tools that will be useful during implementation
5. Identify potential risks or questions

#### 4.3 Write PLANS.md

Write the plan to `PLANS.md` at the project root using the structure template below.

---

## PLANS.md Structure Template

```markdown
# Implementation Plan

**Status:** IN_PROGRESS
**Branch:** feat/PROJ-123-short-description
**Issues:** PROJ-123, PROJ-456
**Created:** YYYY-MM-DD
**Last Updated:** YYYY-MM-DD

## Summary

Brief description of what this plan implements and why.

## Issues

### PROJ-123: Issue Title

**Priority:** High/Medium/Low
**Labels:** Bug, Feature, etc.
**Description:** Copy or summarize the issue description from Linear.

**Acceptance Criteria:**
- [ ] Criterion 1
- [ ] Criterion 2
- [ ] Criterion 3

### PROJ-456: Issue Title

(Same structure for additional issues)

## Prerequisites

List anything that must be true before starting implementation:
- [ ] Database migrations are up to date
- [ ] Environment variables are configured
- [ ] Dependencies are installed

## Implementation Tasks

### Task 1: [Short description]

**Issue:** PROJ-123
**Files:**
- `src/lib/some-module.ts` (create)
- `src/lib/__tests__/some-module.test.ts` (create)

**TDD Steps:**

1. **RED** - Write failing test:
   - Create `src/lib/__tests__/some-module.test.ts`
   - Test that [specific behavior]
   - Run: `npm test -- some-module`
   - Verify: Test fails with [expected error]

2. **GREEN** - Make it pass:
   - Create `src/lib/some-module.ts`
   - Implement [specific logic]
   - Run: `npm test -- some-module`
   - Verify: Test passes

3. **REFACTOR** - Clean up:
   - Extract [shared logic] if needed
   - Ensure naming follows project conventions

**Notes:**
- Use [specific pattern] from existing codebase
- Reference: `src/lib/existing-example.ts`

### Task 2: [Short description]

(Same structure for each task)

### Task N: Integration & Verification

**Issue:** PROJ-123, PROJ-456
**Files:**
- Various files from previous tasks

**Steps:**

1. Run full test suite: `npm test`
2. Run linter: `npm run lint`
3. Run type checker: `npx tsc --noEmit`
4. Manual verification steps:
   - [ ] Step 1
   - [ ] Step 2
5. Build check: `npm run build`

## MCP Usage During Implementation

Document which MCP tools the implementer should use:

| MCP Server | Tool | Purpose |
|------------|------|---------|
| Linear | `update_issue` | Move issues to "In Progress" when starting, "Done" when complete |
| Railway | `list_services` | Check deployment configuration if needed |

## Error Handling

| Error Scenario | Expected Behavior | Test Coverage |
|---------------|-------------------|---------------|
| Invalid input | Return validation error | Unit test |
| Network failure | Retry with backoff | Integration test |
| Auth failure | Redirect to login | E2E test |

## Risks & Open Questions

- [ ] Risk/Question 1: Description and mitigation
- [ ] Risk/Question 2: Description and mitigation

## Scope Boundaries

**In Scope:**
- Items explicitly mentioned in the issues

**Out of Scope:**
- Items NOT part of the current issues
- Future enhancements mentioned but not planned
```

---

## Task Writing Guidelines

When writing tasks in the plan:

1. **Be specific**: Name exact files, functions, and components.
2. **Be ordered**: Each task should build on the previous one. Never reference something that hasn't been created in an earlier task.
3. **Be testable**: Every task must have a clear test-first approach. Write the test assertion before the implementation.
4. **Be small**: Each task should be completable in 15-30 minutes. If it's bigger, break it down further.
5. **Reference patterns**: Point to existing code in the codebase that the implementer should follow.
6. **Include file paths**: Always specify the full file path for every file created or modified.
7. **Include commands**: Provide the exact terminal commands to run tests, linters, etc.
8. **Note dependencies**: If a task depends on a previous task, say so explicitly.

### TDD Pattern

Every implementation task MUST follow the Red-Green-Refactor cycle:

- **RED**: Write a failing test first. Specify what the test asserts and what error message is expected.
- **GREEN**: Write the minimum code to make the test pass. Do not over-engineer.
- **REFACTOR**: Clean up the code while keeping tests green. Extract shared logic, improve naming, etc.

---

## MCP Usage Guidelines

When planning, consider how MCPs will be used during implementation:

### Linear MCP
- Move issues to "In Progress" when implementation starts
- Move issues to "Done" when implementation is complete
- Add comments to issues with progress updates if needed

### Other MCPs
- Check existing services and their configuration
- Verify environment variables are set correctly
- Understand deployment pipeline for the project

---

## Rules

1. **PLANS.md is the single source of truth.** All planning output goes into this file.
2. **Never modify existing code.** This skill only creates `PLANS.md`. It does not create or edit source files.
3. **One plan at a time.** If `PLANS.md` already has an active (non-COMPLETE) plan, do not overwrite it.
4. **Always verify Linear state.** Confirm issues are in Backlog before planning them.
5. **Always read CLAUDE.md.** The project configuration file contains critical context.
6. **Always explore the codebase.** Plans must reference real files and real patterns from the project.
7. **Triage before planning.** Validate every issue against the actual codebase. Cancel issues that don't apply to the current project context.
8. **Use state UUID for Canceled.** Never pass `state: "Canceled"` by name — use the UUID from `list_issue_statuses`. The Linear MCP silently no-ops same-type state transitions by name.
9. **TDD is mandatory.** Every task must follow the Red-Green-Refactor cycle.
10. **Plans must be self-contained.** An implementer should be able to follow the plan without needing to re-read the Linear issues.
11. **Keep scope tight.** Only plan what the issues ask for. Do not add nice-to-haves.
12. **Plans describe WHAT and WHY, not HOW at the code level.** Include: file paths, function names, behavioral specs, test assertions, patterns to follow (reference existing files by path), state transitions. Do NOT include: implementation code blocks, ready-to-paste TypeScript/TSX, full function bodies. The implementer (plan-implement workers) writes all code — your job is architecture and specification. Exception: short one-liners for surgical changes (e.g., "add `if (!session.x)` check after the existing `!session.y` check") are fine.
13. **Move valid issues to Todo.** After writing the plan, update the valid Linear issues to the "Todo" state.
14. **Flag migration-relevant tasks.** If a task changes DB schema, renames columns, changes identity models, renames env vars, or changes session/token formats, add a note in the task: "**Migration note:** [what production data is affected]". The implementer will log this in `MIGRATIONS.md`.

---

## Scope Boundaries

This skill:
- **DOES**: Read Linear issues, explore codebase, read CLAUDE.md, triage issues (cancel invalid ones), write PLANS.md, move valid issues to Todo
- **DOES NOT**: Write source code, write tests, run tests, deploy, create PRs, modify any file other than PLANS.md

If the user asks to also implement the plan, tell them to use the `plan-implement` skill after this one completes.

---

## Error Handling

| Situation | Action |
|-----------|--------|
| Not on `main` branch | STOP. Tell user to switch to main. |
| Uncommitted changes on `main` | STOP. Tell user to commit or stash changes. |
| `PLANS.md` has active plan | STOP. Tell user to complete or remove the existing plan. |
| Linear issue not found | STOP. Tell user the issue ID is invalid. |
| Linear issue not in Backlog | WARN user but continue if they confirm. |
| No CLAUDE.md found | WARN user. Continue with reduced context. |
| MCP server unavailable | WARN user. Continue without that MCP's context. |
| User specifies no issues | Fetch ALL Backlog issues and plan them (default behavior). |
| All issues invalid after triage | STOP. Cancel all issues, inform user no plan needed. |
| Some issues invalid after triage | Cancel invalid issues, plan only valid ones. |

---

## Termination: Git Workflow

After writing `PLANS.md` and moving issues to Todo in Linear, complete the session with these git operations:

1. **Create a feature branch:**
   ```bash
   git checkout -b feat/PROJ-123-short-description
   ```
   Use the primary issue key in the branch name. If multiple issues, use the first one.

2. **Stage and commit the plan** (no `Co-Authored-By` tags):
   ```bash
   git add PLANS.md
   git commit -m "plan(PROJ-123): add implementation plan for [short description]

   Issues: PROJ-123, PROJ-456
   Status: Todo in Linear"
   ```

3. **Push the branch:**
   ```bash
   git push -u origin feat/PROJ-123-short-description
   ```

4. **Report completion** to the user with:
   - Branch name
   - Summary of what was planned
   - Number of tasks in the plan
   - Next step: use `plan-implement` to start implementation
