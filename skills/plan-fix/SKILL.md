---
name: plan-fix
description: Investigates bugs AND creates actionable TDD fix plans. Creates Linear issues in Todo state. Use when you know you want to fix something - user reports errors, deployment failures, wrong data, or UI issues. Can be chained from investigate skill. Discovers MCPs from CLAUDE.md for debugging (logs, etc.).
argument-hint: <bug description>
allowed-tools: Read, Edit, Write, Glob, Grep, Task, Bash, mcp__linear__list_teams, mcp__linear__list_issues, mcp__linear__get_issue, mcp__linear__create_issue, mcp__linear__update_issue, mcp__linear__list_issue_labels, mcp__linear__list_issue_statuses, mcp__sentry__find_organizations, mcp__sentry__find_projects, mcp__sentry__search_issues, mcp__sentry__get_issue_details, mcp__sentry__analyze_issue_with_seer, mcp__sentry__search_events, mcp__sentry__search_issue_events, mcp__sentry__get_issue_tag_values
disable-model-invocation: true
---

Investigate bugs and create TDD fix plans in PLANS.md. Creates Linear issues in Todo state.

## 1. Git Pre-flight Check

Before starting any investigation, verify git status:

```bash
git branch --show-current
git status --porcelain
```

- **STOP if NOT on `main` branch.** Tell the user: "Not on main branch. Please switch to main before planning: `git checkout main`"
- **STOP if there are uncommitted changes.** Tell the user to commit or stash first.
- **Check if behind remote:** `git fetch origin && git status -uno` — STOP if behind.

## 2. PLANS.md Pre-flight

Check if `PLANS.md` already exists at the project root:

- If it does not exist: OK, you will create it when documenting findings.
- If it exists with `Status: COMPLETE`: OK, overwrite with new fix plan.
- If it exists with active (non-COMPLETE) content: **STOP.** Tell the user there is an active plan that must be completed or removed first.
- In all cases, check for an existing section about this bug to avoid duplicates.

## 3. Verify Linear MCP

Call `mcp__linear__list_teams`. If unavailable, **STOP** and tell the user: "Linear MCP is not connected. Run `/mcp` to reconnect, then re-run this skill."

## 4. Read Project Context

Read `CLAUDE.md` at the project root (if it exists) to understand:
- Project structure and conventions
- Available MCPs (Linear, Railway, etc.)
- Tech stack details
- Testing conventions
- Any project-specific debugging notes

**Discover team name:** Look for LINEAR INTEGRATION section in CLAUDE.md. If not found, use `mcp__linear__list_teams` to discover the team name dynamically. Store the discovered team name for use throughout the skill.

## 5. Classify Bug Type

Categorize the reported issue into one of these types:

| Category | Description | Key Investigation Areas |
|----------|-------------|------------------------|
| **API Error** | Backend route failures, 500s, bad responses | Route handlers, middleware, external API calls, error handling |
| **Auth Issue** | Login failures, session problems, unauthorized access | Auth configuration, session management, middleware, token handling |
| **Deployment Failure** | Build errors, runtime crashes on Railway | Build logs, environment variables, dependency issues |
| **Frontend Bug** | UI rendering issues, broken interactions, wrong data display | React components, state management, data fetching, hydration |
| **Data Issue** | Wrong data, missing data, data corruption | Database queries, API transformations, caching, race conditions |
| **Performance** | Slow responses, timeouts, memory issues | Query performance, bundle size, API response times |
| **Integration** | Third-party service failures (AI APIs, payment, etc.) | API keys, request/response formats, rate limits, error handling |

## 6. Gather Evidence

### 6.1 Codebase Investigation

Search the codebase for relevant code using dedicated tools (NOT Bash):

- **Use Glob** to find files by pattern
- **Use Grep** to search for relevant patterns: function names, error messages, class references
- **Use Read** to examine source files, configs, and build scripts

What to find:
- The files involved in the bug
- Trace the code path from entry point to the error
- Look for recent changes that might have introduced the bug
- Check test files for related test coverage

### 6.2 Deployment Logs (if MCP available)

If CLAUDE.md lists deployment MCPs (e.g., Railway MCP) and the bug involves deployment or runtime errors, use the MCP to check logs:

- Check recent deployment status
- Look for error logs around the time of the reported issue
- Check environment variable configuration (without exposing values)
- Review build logs for warnings or errors

### 6.3 Linear Context

Search Linear for related issues:

- Use `mcp__linear__list_issues` to find existing issues about this bug
- Check if there are related issues that provide context
- Look for previously attempted fixes

### 6.4 Sentry Context

If the bug involves production crashes, errors, or runtime issues, search Sentry for related issues. Use ToolSearch to load Sentry tools before calling them.

1. **Find the org/project** — Use `mcp__sentry__find_organizations` then `mcp__sentry__find_projects` to get slugs
2. **Search for issues** — Use `mcp__sentry__search_issues` with natural language (e.g., "unresolved crashes from last week")
3. **Get issue details** — Use `mcp__sentry__get_issue_details` for full stack traces and metadata
4. **Analyze root cause** — Use `mcp__sentry__analyze_issue_with_seer` for AI-powered analysis
5. **Check distributions** — Use `mcp__sentry__get_issue_tag_values` for environment/release breakdown

If Sentry issues are found:
- Document the Sentry issue ID and URL in the PLANS.md `**Sentry:**` field
- Note frequency, affected users, and releases in Evidence section
- Include the Sentry issue reference in the Linear issue description (see Section 8)

### 6.5 Reproduce the Issue

When possible, try to reproduce:

```bash
# Check if tests exist and if they catch the issue
npm test 2>&1 | tail -50

# Check for TypeScript errors
npx tsc --noEmit 2>&1 | tail -50

# Check for lint errors
npm run lint 2>&1 | tail -50
```

## 7. Document Findings in PLANS.md

Read `references/plans-template.md` for the complete template.

**Source field:** `Bug report: [Summary of $ARGUMENTS]`

Include: Context Gathered (Codebase Analysis + MCP Context + Investigation), Tasks, Post-Implementation Checklist, Plan Summary.
Omit: Triage Results subsection.

The Investigation subsection under Context Gathered must include: bug report, classification (type/severity/affected area), root cause, evidence (file paths with line numbers — no code blocks), and impact.

## 8. Create Linear Issue

Create a Linear issue in the discovered team with status "Todo":

1. First, get the team statuses to find the "Todo" state ID:
   ```
   mcp__linear__list_issue_statuses for team [discovered team name]
   ```

2. Get available labels:
   ```
   mcp__linear__list_issue_labels for team [discovered team name]
   ```

3. Create the issue:
   ```
   mcp__linear__create_issue with:
   - team: [Discovered team name]
   - title: "[Bug Type] Brief description of the fix needed"
   - description: |
     ## Bug Report
     [Summary of the issue]

     ## Sentry Issue (if applicable)
     [Sentry issue URL] — [event count] events, [user count] users, release [version]
     **Action:** Resolve this Sentry issue after fix is merged and released.

     ## Root Cause
     [What was found during investigation]

     ## Fix Plan
     See PLANS.md for detailed TDD fix plan.

     ## Files Affected
     - `path/to/file.ts`
     - `path/to/another-file.ts`

     ## Acceptance Criteria
     - [ ] Failing test written and passes after fix
     - [ ] All existing tests pass
     - [ ] No TypeScript errors
     - [ ] Deployed successfully
     - [ ] Sentry issue resolved (if applicable)
   - status: "Todo"
   - Apply relevant labels (bug, etc.)
   ```

   Omit the "Sentry Issue" section if the bug did not originate from Sentry.

4. Update PLANS.md with the created issue key (PROJ-xxx).

## 9. Error Handling

| Situation | Action |
|-----------|--------|
| Cannot reproduce the bug | Document what was tried, create issue with "needs-reproduction" label |
| Root cause unclear | Document hypotheses ranked by likelihood, create issue with investigation notes |
| Multiple bugs found | Create separate PLANS.md sections and Linear issues for each |
| Bug is in a dependency | Document the dependency issue, check for updates/workarounds, note in issue |
| Deployment MCP unavailable | Skip deployment log analysis, note in findings |
| Linear MCP unavailable | Document the issue details in PLANS.md only, tell user to create manually |
| CLAUDE.md not found | Proceed with standard Next.js conventions |
| Existing fix in progress | Check the existing Linear issue and PLANS.md entry, update rather than duplicate |
| Bug is actually a feature request | Reclassify and suggest using add-to-backlog skill instead |

## 10. Rules

- **NEVER modify application code.** This skill only investigates and plans.
- **NEVER run destructive commands** (no `rm`, no `git reset --hard`, no database mutations).
- **ALWAYS use TDD approach** in fix plans - tests first, then implementation.
- **ALWAYS check for existing Linear issues** before creating new ones to avoid duplicates.
- **ALWAYS include file paths and line numbers** in evidence and fix plans.
- **ALWAYS propose a branch name** following the pattern `fix/PROJ-xxx-brief-description`.
- **Discover MCPs from CLAUDE.md** - don't hardcode MCP names or paths
- **Keep fix plans actionable** - another developer (or AI agent) should be able to follow the plan without additional context.
- **Severity guidelines:**
  - **Critical:** Production down, data loss, security vulnerability
  - **High:** Feature broken for all users, significant data issues
  - **Medium:** Feature partially broken, workaround exists
  - **Low:** Minor UI issue, edge case, cosmetic problem
- **DO NOT expose secrets, API keys, or sensitive environment variable values** in PLANS.md or Linear issues.
- **DO NOT hallucinate code** - only reference code that actually exists in the codebase.
- **Plans describe WHAT and WHY, not HOW at the code level.** Include: file paths, function names, behavioral specs, test assertions, patterns to follow (reference existing files by path), state transitions. Do NOT include: implementation code blocks, ready-to-paste TypeScript/TSX, full function bodies. The implementer (plan-implement workers) writes all code — your job is architecture and specification. Exception: short one-liners for surgical changes (e.g., "add `if (!session.x)` check after the existing `!session.y` check") are fine.
- **Flag migration-relevant fixes** — If the fix changes DB schema, renames columns, changes identity models, renames env vars, or changes session/token formats, add a note in the fix plan: "**Migration note:** [what production data is affected and how to migrate]". The plan MUST include a migration strategy (e.g., startup detection of old format + automatic migration). The implementer will log this in `MIGRATIONS.md`.

## 11. Scope Boundaries

This skill is specifically for:
- Investigating reported bugs and errors
- Creating structured fix plans with TDD approach
- Creating Linear issues for tracking

This skill is NOT for:
- Actually implementing fixes (use plan-implement for that)
- Adding new features (use plan-backlog or add-to-backlog)
- Code reviews (use code-audit)
- General investigation without a fix intent (use investigate)
- Refactoring (create a separate task)

## 12. Termination

Follow the termination procedure in `references/plans-template.md`: output the Plan Summary, then create branch, commit (no `Co-Authored-By` tags), and push.

If chained from investigate skill, reference the investigation findings and note any additional evidence found during planning.

Do not ask follow-up questions. Do not offer to implement. Output the summary and stop.
