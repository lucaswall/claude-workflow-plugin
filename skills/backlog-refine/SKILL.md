---
name: backlog-refine
description: Refine vague Linear Backlog issues into well-specified, actionable items. Use when user says "refine backlog", "refine PROJ-123", "improve backlog items", or "clarify issues". Fetches Backlog issues, analyzes gaps, asks clarifying questions, and updates Linear after user confirms.
argument-hint: [PROJ-123 PROJ-124 or blank for picker]
allowed-tools: Read, Glob, Grep, AskUserQuestion, mcp__linear__list_teams, mcp__linear__list_issues, mcp__linear__get_issue, mcp__linear__update_issue, mcp__linear__create_issue, mcp__linear__create_comment, mcp__linear__list_issue_labels, mcp__linear__list_issue_statuses
disable-model-invocation: true
---

Refine vague Backlog issues into well-specified, actionable items through interactive discussion with the user.

## Pre-flight

1. **Verify Linear MCP** — Call `mcp__linear__list_teams`. If unavailable, STOP and tell the user: "Linear MCP is not connected. Run `/mcp` to reconnect, then re-run this skill."
2. **Read CLAUDE.md** — Load project conventions for context. Extract the team name from the LINEAR INTEGRATION section (look for "Team:" line). If CLAUDE.md doesn't exist or doesn't have a LINEAR INTEGRATION section, use the team name from `mcp__linear__list_teams` (there should only be one team in most cases).

## Input Handling

### Mode 1: Specific Issues ($ARGUMENTS contains issue identifiers)

Parse issue identifiers from `$ARGUMENTS` (e.g., `PROJ-123`, `PROJ-124`).
- Fetch each issue using `mcp__linear__get_issue`
- Continue to Analysis phase

### Mode 2: No Arguments (picker mode)

If `$ARGUMENTS` is empty or doesn't contain issue identifiers:
1. Fetch all Backlog issues: `mcp__linear__list_issues` with `team: "<discovered-team-name>"`, `state: "Backlog"`, `includeArchived: false`
2. Score each issue's refinement readiness (see Refinement Score below)
3. Assess each issue's drop likelihood (see Drop Assessment below)
4. Display a markdown table with columns: #, Issue, Title, Priority, Labels, Score, Drop?
5. Present to user using `AskUserQuestion`:
   - Include the score and drop indicator in each option's description (e.g., "Screen to show food logged by date — Feature — Score: 3/5 — Drop? No")
   - Let user pick which issues to refine
6. Fetch full details for selected issues

### Refinement Score

Rate each issue 1–5 based **only** on the data returned from `list_issues` (title, description, priority, labels). Do NOT read source files or fetch full details for scoring — this is a quick triage.

| Score | Meaning | Criteria |
|-------|---------|----------|
| 5 | Ready | Has problem statement, file paths, impact, acceptance criteria, correct priority/labels |
| 4 | Minor gaps | Missing one of: acceptance criteria, impact statement, or implementation hints |
| 3 | Needs work | Has file paths and problem description but missing 2+ of: acceptance criteria, impact, specificity |
| 2 | Unclear | Very short description, no file paths, scope ambiguous |
| 1 | Stub | Title only or single sentence, no useful detail |

**Scoring checklist** (deduct 1 point from 5 for each gap):
- Description is ≤1 sentence with no specifics → −1
- No acceptance criteria or definition of done → −1
- No affected files/routes/components mentioned in the description → −1
- Scope is ambiguous (could mean multiple things) → −1
- Priority or labels seem mismatched with description → −1

**Truncation rule:** `list_issues` truncates descriptions at ~300-400 chars, which systematically hides acceptance criteria (they come last in the standard template). When the visible description uses the structured template format (`**Problem:**`, `**Context:**`, `**Impact:**`, `**Fix:**` section headers) and is truncated, **assume acceptance criteria are present** — do NOT deduct for them. Only deduct for acceptance criteria when the full description is visible (short enough to not be truncated) and they're genuinely missing.

Minimum score is 1. Issues scoring 5 can still be selected — refinement may find minor improvements after reading full details and source code.

### Drop Assessment

For each issue, make a quick gut-feel assessment of whether it should be dropped. This is based **only** on `list_issues` data (title, description, priority, labels) — no codebase reading at this stage.

| Indicator | Meaning | Criteria |
|-----------|---------|----------|
| **Likely** | Probably should cancel | Title describes a theoretical concern, "nice to have" with no clear impact, duplicates another issue, or addresses a non-problem for this project context (single-user, behind auth, etc.) |
| **Maybe** | Worth discussing | Unclear value, scope seems disproportionate to benefit, or might be outdated |
| **No** | Keep it | Clear problem, real impact, actionable |

The Drop? column helps the user prioritize which issues to examine first — issues marked "Likely" may not need refinement at all, just cancellation.

## Analysis Phase

For each issue, analyze and identify gaps. Read relevant source files referenced in the issue description to understand context.

Check for these common problems in Backlog issues:

### Vagueness Checks
- **Missing problem statement** — Does the description explain what's wrong or missing?
- **Missing context** — Are affected files, routes, or components identified?
- **Missing impact** — Is it clear why this matters (user impact, data quality, errors)?
- **Missing acceptance criteria** — How would you know this is done?
- **Scope too broad** — Should this be split into multiple focused issues?
- **Scope too narrow** — Is this really a sub-task of a larger issue?

### Quality Checks
- **Wrong priority** — Does the priority match the actual impact?
- **Wrong/missing labels** — Does the label correctly categorize the issue?
- **Duplicate or overlapping** — Does another Backlog issue cover the same ground?
- **Outdated** — Has the referenced code changed since the issue was created?
- **Missing implementation hints** — Could helpful pointers be added for `plan-backlog`?

### Codebase Cross-Reference
- If the issue mentions specific files/areas, read them to verify the problem still exists
- If the issue is vague about location, search the codebase to identify affected areas
- Note any related code patterns that provide useful context

## Discussion Phase

For each issue, present your analysis to the user:

```
## PROJ-123: [Current Title]

**Current state:**
[Brief summary of what the issue says now]

**Issues found:**
- [Gap 1: e.g., "No acceptance criteria — unclear when this is done"]
- [Gap 2: e.g., "Description says 'improve error handling' but doesn't specify which errors"]
- [Gap 3: e.g., "Priority is Low but this is a security concern — should be High"]

**Suggested improvements:**
- [Suggestion 1: e.g., "Add specific error scenarios to handle"]
- [Suggestion 2: e.g., "Split into two issues: API errors vs UI errors"]
- [Suggestion 3: e.g., "Bump priority to High (Security label)"]

**Questions for you:**
- [Question 1: e.g., "Should this cover both API and UI error handling, or just API?"]
- [Question 2: e.g., "Is there a specific error you've seen that triggered this issue?"]
```

Then engage in a back-and-forth discussion:
- Ask clarifying questions to fill gaps
- Suggest concrete improvements based on codebase knowledge
- Propose title rewording if current title is vague
- Recommend priority/label changes if warranted
- Suggest splitting if scope is too broad (see Splitting Issues below)
- Flag if the issue might be outdated or already fixed
- **Suggest cancellation** if the issue shouldn't be done (see Suggesting Cancellation below)

Continue the discussion until the user says they're done or confirms the refinements.

### Splitting Issues

When an issue covers multiple distinct problems or its scope is too broad, suggest splitting it. During discussion, propose the split clearly:

```
**Suggested split for PROJ-123:**
This issue covers both API error handling and UI error display. I'd suggest splitting:

1. **PROJ-123 (updated):** "API routes return generic 500 on upload failures"
   - Label: Bug, Priority: High
2. **New issue:** "Upload UI shows no feedback on server errors"
   - Label: Improvement, Priority: Medium
```

If the user agrees to a split:
- The **original issue** (PROJ-123) is updated to become the first split item (new title, description, priority, labels)
- **New issues** are created via `mcp__linear__create_issue` for the remaining split items, all in `state: "Backlog"` with proper labels and priorities
- Each split issue gets a full refined description (using the standard format)

### Suggesting Cancellation

During analysis, if the issue shouldn't be done, suggest canceling it. Valid reasons for cancellation:

- **Problem doesn't exist** — The referenced code doesn't have the described issue
- **Already fixed** — The problem was resolved by other work
- **Not applicable** — Theoretical concern that doesn't apply (e.g., multi-user issue for a single-user app)
- **Cost exceeds value** — The effort to fix is disproportionate to the benefit
- **Duplicate** — Another issue already covers this
- **Outdated** — The referenced code or feature no longer exists

Present cancellation suggestions clearly during discussion:

```
## PROJ-123: [Current Title]

**Recommendation: Cancel this issue**

**Reasons:**
- [Reason 1: e.g., "This is a single-user app behind auth — the XSS vector described requires a second user"]
- [Reason 2: e.g., "The referenced code in src/lib/foo.ts was removed in PR #30"]

**Context:** [What you found when reading the codebase that supports this recommendation]
```

If the user agrees to cancel, the issue will be handled in the Update Phase.

## Update Phase

When the user confirms they're done refining:

1. **Show the update preview** — Present the final version of each issue:

```
## Updates to apply:

### PROJ-123 (update)
- **Title:** [Original] → [New title, if changed]
- **Description:** [Full new description]
- **Priority:** [Original] → [New, if changed]
- **Labels:** [Original] → [New, if changed]

### New issue (split from PROJ-123)
- **Title:** [Title]
- **Description:** [Full description]
- **Priority:** [Priority]
- **Labels:** [Labels]

### PROJ-124 (cancel)
- **Reason:** [Specific reason for cancellation]

### PROJ-125 (update)
...
```

2. **Wait for confirmation** — Ask user to confirm before updating.

3. **Apply updates:**
   - Use `mcp__linear__update_issue` for each updated issue (title, description, priority, labels)
   - Use `mcp__linear__create_issue` for each new split issue with `team: "<discovered-team-name>"`, `state: "Backlog"`, and proper labels/priority
   - For canceled issues, follow the Cancellation Procedure below
   - For kept issues when other issues in the same session were canceled, follow the Vetted Marker below

### Cancellation Procedure

For each issue the user agreed to cancel:

1. **Add a comment** explaining the cancellation reason:

```
mcp__linear__create_comment(issueId: "PROJ-xxx", body: "Canceled during refinement: [reason]")
```

The reason should be specific, e.g.:
- "Problem doesn't exist — the referenced code in src/lib/foo.ts was removed in PR #30."
- "Single-user app behind auth — the described XSS vector requires a second user."
- "Cost exceeds value — would require rewriting the entire module for a marginal improvement."

2. **Move to Canceled state.**

**CRITICAL: Linear MCP same-type state bug.** "Duplicate" and "Canceled" are both `type: canceled` in Linear. Passing `state: "Canceled"` by name silently no-ops if the issue is already in another canceled-type state. To reliably cancel issues, first fetch the team's statuses to get the Canceled state UUID:

```
mcp__linear__list_issue_statuses(team: "<discovered-team-name>")
```

Find the status with `name: "Canceled"` and use its `id` (UUID) in the update call:

```
mcp__linear__update_issue(id: "PROJ-xxx", state: "<canceled-state-uuid>")
```

**Always use the UUID, never the name**, for canceled-type state transitions.

### Vetted Marker

When a refinement session cancels some issues but keeps others, the kept issues must be marked as vetted so that downstream skills (plan-backlog) don't re-question or drop them.

**Prepend this note** to the top of the kept issue's description (before `**Problem:**`):

```
**Refinement:** Reviewed YYYY-MM-DD — confirmed valid. [1-sentence summary of why it was kept]. Do not drop.
```

This note is added via `mcp__linear__update_issue` along with any other description changes. If the kept issue's description doesn't need other changes, still update it to add the Refinement note.

### Refined Description Format

Use this structure for updated descriptions:

```
**Problem:**
[Clear, specific problem statement — 1-2 sentences]

**Context:**
[Affected files, routes, or components — be specific]

**Impact:**
[Why this matters — user-facing impact, data quality, security, etc.]

**Acceptance Criteria:**
- [ ] [Specific, verifiable criterion]
- [ ] [Another criterion]

**Implementation Hints:** (optional)
[Pointers for plan-backlog: patterns to follow, related code, constraints]
```

## Processing Multiple Issues

When refining multiple issues:
- Process one at a time to keep discussion focused
- After completing discussion on one issue, move to the next
- At the end, show all updates together for a single confirmation

## Error Handling

| Situation | Action |
|-----------|--------|
| Linear MCP not connected | STOP — tell user to run `/mcp` |
| Issue ID not found | Tell user, skip that issue |
| No Backlog issues exist | Tell user "No Backlog issues found" and stop |
| User picks no issues | Stop gracefully |
| Referenced code doesn't exist anymore | Note as potentially outdated in analysis |
| Issue already well-specified | Tell user it looks good, suggest minor tweaks if any |
| All selected issues canceled | Apply cancellations and stop — no refinement needed |

## Rules

- **Discussion-driven** — Always engage the user, don't auto-refine silently
- **Confirm before updating** — Show preview of all changes before applying
- **Preserve user intent** — Refine the issue, don't change its fundamental purpose
- **One issue at a time** — Keep discussion focused during analysis
- **No code changes** — This skill only updates Linear issues

## Termination

After applying updates, output:

```
Refinement complete.

Updated X issues:
- PROJ-123: [New title] — [brief summary of changes]
- PROJ-124: [New title] — [brief summary of changes]

Created Y issues (from splits):
- PROJ-130: [Title] (split from PROJ-123) — [Label, Priority]
- PROJ-131: [Title] (split from PROJ-124) — [Label, Priority]

Canceled Z issues:
- PROJ-126: [Title] — [brief cancellation reason]

Unchanged:
- PROJ-125: Already well-specified

Next step: Use `plan-backlog PROJ-123` to create an implementation plan.
```

Do not ask follow-up questions after termination.
