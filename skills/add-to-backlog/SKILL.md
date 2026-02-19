---
name: add-to-backlog
description: Add issues to Linear Backlog from free-form input. Use when user says "add to backlog", "create backlog issues", "track this", or describes tasks/improvements/bugs to add. Interprets user's ideas, investigation findings, or conversation context into well-structured Backlog issues. Can process multiple items at once.
argument-hint: [description of what to add, or "from conversation", or "from investigation"]
allowed-tools: Read, Glob, Grep, Task, mcp__linear__list_teams, mcp__linear__list_issues, mcp__linear__get_issue, mcp__linear__create_issue, mcp__linear__list_issue_labels, mcp__linear__list_issue_statuses
disable-model-invocation: true
---

Add issues to Linear Backlog from user input. Interprets free-form descriptions into well-structured issues.

## Purpose

- Convert user's free-form ideas into structured Backlog issues
- Parse multiple items from a single input
- Reference conversation context or investigation findings
- Write problem-focused descriptions (what, not how)
- Include implementation hints for `plan-backlog` to use later

## Input Modes

The skill supports three input modes based on $ARGUMENTS:

### Mode 1: Direct Description
User provides task descriptions directly:
```
/add-to-backlog Add rate limiting to API routes, also need to handle image upload errors gracefully, and the barcode scanner fails on blurry images
```

### Mode 2: From Conversation
User references the current conversation:
```
/add-to-backlog from conversation - add the three issues we discussed
/add-to-backlog add all the improvements mentioned above
/add-to-backlog track the bug we just found
```

### Mode 3: From Investigation
User references findings from `investigate` skill:
```
/add-to-backlog from investigation findings
/add-to-backlog add the issues found by investigate
```

## Pre-flight

**Verify Linear MCP:** Call `mcp__linear__list_teams`. If unavailable, **STOP** and tell the user: "Linear MCP is not connected. Run `/mcp` to reconnect, then re-run this skill."

**Discover team name:** Read CLAUDE.md and look for LINEAR INTEGRATION section. Extract the team name from patterns like:
- "Team: 'ProjectName'"
- "Team: ProjectName"

If CLAUDE.md doesn't have a LINEAR INTEGRATION section, call `mcp__linear__list_teams` to discover the team name dynamically.

Store the discovered team name in a variable for use throughout the skill.

## Workflow

1. **Parse input** - Understand what to add based on $ARGUMENTS
2. **Identify items** - Separate multiple items from the input
3. **Check existing Backlog** - Avoid duplicates
4. **Draft issues** - Write problem-focused descriptions
5. **Create in Linear** - Add to Backlog state

## Issue Structure

Each issue should have:

### Title
- Clear, concise problem statement
- Action-oriented: "Rate limiting missing on API routes", "Barcode scanner fails on blurry images"
- NO solution in title

### Description
Structure:
```
**Problem:**
[What is wrong or missing - 1-2 sentences]

**Context:**
[Where this occurs, affected files/areas - brief]

**Impact:**
[Why this matters - user impact, data quality, errors]

**Implementation Hints:** (optional)
[Suggestions for plan-backlog, patterns to follow, related code]
```

### Labels
Map to Linear labels based on issue type:

| Issue Type | Linear Label |
|------------|--------------|
| Missing functionality | Feature |
| Broken behavior | Bug |
| Better approach exists | Improvement |
| Code quality issue | Technical Debt |
| Security concern | Security |
| Slow/resource issue | Performance |
| Style/format issue | Convention |

### Priority
Assess based on impact:

| Impact | Priority |
|--------|----------|
| Data loss, security hole, production down | 1 (Urgent) |
| Incorrect data, broken feature | 2 (High) |
| Inconvenience, missing enhancement | 3 (Medium) |
| Minor polish, nice-to-have | 4 (Low) |

## Parsing Input

### Direct Descriptions
Look for natural separators:
- "also", "and also", "additionally"
- Numbered lists: "1.", "2.", etc.
- Bullet points: "-", "*"
- Commas followed by action verbs
- Complete sentences as separate items

Example:
```
"Add rate limiting, also handle upload errors, and fix the blurry barcode bug"
```
--> Three issues:
1. Rate limiting missing on API routes
2. Image upload error handling incomplete
3. Barcode scanner fails on blurry images

### Conversation References
When user says "from conversation" or similar:
1. Review the conversation above
2. Identify discussed problems, improvements, or bugs
3. Extract actionable items

### Investigation References
When user mentions investigation findings:
1. Look for investigation output in conversation
2. Extract issues, errors, or recommendations found
3. Convert findings into actionable issues

## Duplicate Detection

Before creating, check existing Backlog:
1. Query `mcp__linear__list_issues` with `team=[discovered team name], state=Backlog, includeArchived=false`
2. Compare proposed issues against existing titles/descriptions
3. If similar issue exists:
   - Skip the duplicate automatically

## Creating Issues

Use `mcp__linear__create_issue` for each issue (skip duplicates automatically):

```
team: [Discovered team name]
state: "Backlog"
title: "[Issue title]"
description: "**Problem:**\n[description]\n\n**Context:**\n[context]\n\n**Impact:**\n[impact]\n\n**Implementation Hints:**\n[hints]"
priority: [1|2|3|4]
labels: [Mapped label]
```

## Writing Good Issues

### DO:
- Focus on the problem, not the solution
- Include context about where/when the issue occurs
- Explain impact to help prioritization
- Add implementation hints for plan-backlog
- Reference related files or code if known

### DON'T:
- Include step-by-step implementation
- Write the solution in the description
- Use vague language ("improve this", "fix the thing")
- Create issues without clear problem statement

### Good Example:
```
**Problem:**
API routes have no rate limiting, allowing unlimited requests from any client.

**Context:**
Affects all routes under app/api/. Public endpoints like /api/scan and /api/food are most vulnerable.

**Impact:**
Service can be overwhelmed by automated requests, causing downtime for all users.

**Implementation Hints:**
- Consider upstash/ratelimit or next-rate-limit
- See existing middleware pattern in middleware.ts
- Should return 429 with Retry-After header
```

### Bad Example:
```
Add rate limiting. Use upstash ratelimit library. Create a middleware that checks IP address and limits to 100 requests per minute. Return 429 status code.
```

## Error Handling

| Situation | Action |
|-----------|--------|
| $ARGUMENTS empty | Ask user what to add |
| Can't parse items | Show interpretation, ask for clarification |
| Linear unavailable | Stop, tell user to check Linear auth |
| All items are duplicates | Report existing issues, skip creation |
| Conversation reference unclear | List recent topics, ask which to add |

## Rules

- **Problem-focused** - Describe what's wrong, not how to fix
- **Include hints** - Help plan-backlog with implementation suggestions
- **Check duplicates** - Avoid cluttering backlog
- **One problem per issue** - Split combined issues

## Termination

After creating issues, output:

```
Created X issues in Linear Backlog:

- PROJ-123: [Title] (Label, Priority)
- PROJ-124: [Title] (Label, Priority)
- PROJ-125: [Title] (Label, Priority)

Skipped:
- [Description] - duplicate of PROJ-12

Next steps:
- Review issues in Linear Backlog
- Use `plan-backlog` to create implementation plans
- Use `plan-backlog PROJ-123` to plan a specific issue
```

Do not ask follow-up questions. Do not offer to plan or implement.
