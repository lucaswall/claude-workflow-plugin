---
name: code-audit
description: Audits codebase using an agent team with 3 domain-specialized reviewers (security, reliability, quality). Creates Linear issues in Backlog state for findings. Use when user says "audit", "find bugs", "check security", "review codebase", or "team audit". Higher token cost, faster and deeper analysis. Falls back to single-agent mode if agent teams unavailable.
argument-hint: [optional: specific area like "lib" or "api"]
allowed-tools: Read, Glob, Grep, Task, Bash, TeamCreate, TeamDelete, SendMessage, TaskCreate, TaskUpdate, TaskList, TaskGet, mcp__linear__list_teams, mcp__linear__list_issues, mcp__linear__get_issue, mcp__linear__create_issue, mcp__linear__update_issue, mcp__linear__list_issue_labels, mcp__linear__list_issue_statuses
disable-model-invocation: true
---

Perform a comprehensive code audit using an agent team with domain-specialized reviewers. You are the **team lead/coordinator**. You orchestrate 3 reviewer teammates who scan the codebase in parallel, then you merge findings and create Linear issues.

**If agent teams are unavailable** (TeamCreate fails), fall back to single-agent mode — see "Fallback: Single-Agent Mode" section.

## Pre-flight

1. **Verify Linear MCP** — Call `mcp__linear__list_teams`. If unavailable, STOP and tell the user: "Linear MCP is not connected. Run `/mcp` to reconnect, then re-run this skill."
2. **Discover project info from CLAUDE.md** — Read the LINEAR INTEGRATION section to find:
   - Team name (e.g., "Food Scanner")
   - Issue prefix (e.g., PROJ-xxx)
   - If LINEAR INTEGRATION section not found, call `mcp__linear__list_teams` to discover the team
3. **Read CLAUDE.md** — Load project-specific rules to audit against (if exists)
4. **Query Linear Backlog** — Get existing issues using `mcp__linear__list_issues` with:
   - `team`: [discovered team name]
   - `state`: "Backlog"
   - `includeArchived`: false
   - For each issue, record: ID, title, labels, priority, description
   - **Audit issues** (labels: Bug, Security, Performance, Convention, Technical Debt) → mark as `pending_validation`
   - **Non-audit issues** (labels: Feature, Improvement) → mark as `preserve` (skip validation)
5. **Discover project structure** — Read `tsconfig.json`, `package.json`, `.gitignore` in parallel
   - Use Glob with patterns from tsconfig.json `include` to identify source directories
   - If no tsconfig, use conventions: `src/`, `lib/`, `app/`, `packages/`
6. **Run `npm audit`** — Capture critical/high dependency vulnerabilities for later

## Team Setup

### Create the team

Use `TeamCreate`:
- `team_name`: "code-audit"
- `description`: "Parallel code audit with domain-specialized reviewers"

**If TeamCreate fails**, switch to Fallback: Single-Agent Mode (see below).

### Create tasks

Use `TaskCreate` to create 3 review tasks (these track progress for each reviewer):

1. **"Security audit"** — Security & auth review of the codebase
2. **"Reliability audit"** — Bugs, async, resources, memory leaks, timeouts
3. **"Quality audit"** — Type safety, conventions, logging, tests, dead code

### Spawn 3 reviewer teammates

Use the `Task` tool with `team_name: "code-audit"`, `subagent_type: "general-purpose"`, and `model: "sonnet"` to spawn each reviewer. Give each a `name` and a detailed `prompt` (see Reviewer Prompts below).

Spawn all 3 reviewers in parallel (3 concurrent Task calls in one message).

**IMPORTANT:** Each reviewer prompt MUST include:
- Their specific domain checklist (copied from the Reviewer Prompts section)
- The focus area if `$ARGUMENTS` specifies one
- The list of existing `pending_validation` issues relevant to their domain (so they can validate them)
- Instructions to report findings as a structured message to the lead

### Assign tasks

After spawning, use `TaskUpdate` to assign each task to its reviewer by name.

## Reviewer Prompts

Each reviewer gets a tailored prompt. Include the full text below in each reviewer's spawn prompt, substituting the domain-specific section.

### Common Preamble (include in ALL reviewer prompts)

```
You are a code audit reviewer for this project. Your job is to scan the ENTIRE codebase and find issues in your assigned domain.

RULES:
- Analysis only — do NOT modify any source code
- Do NOT create Linear issues — report findings to the team lead
- No solutions — document problems only, not fixes
- Be specific — include file paths and approximate line numbers
- Be thorough — check every file in scope
- Focus area: {$ARGUMENTS or "entire codebase"}

WORKFLOW:
1. Read CLAUDE.md for project-specific rules
2. Read .claude/skills/code-audit/references/compliance-checklist.md for detailed audit checks in your domain
3. Discover all source files using Glob (check tsconfig.json include patterns)
4. Read each source file systematically
5. Use Grep to search for specific patterns (see your checklist AND the compliance checklist)
6. Validate any existing issues assigned to you (check if code still has the problem)
7. When done, send your findings to the lead using SendMessage

EXISTING ISSUES TO VALIDATE:
{list of pending_validation issues relevant to this reviewer's domain}
For each, check if the referenced code still has the problem. Report as:
- FIXED: [issue ID] - [reason]
- STILL EXISTS: [issue ID]

FINDINGS FORMAT - Send a message to the lead with this structure:
---
DOMAIN: {domain name}
VALIDATED EXISTING ISSUES:
- FIXED: PROJ-XX - [reason]
- STILL EXISTS: PROJ-YY

NEW FINDINGS:
1. [category-tag] [priority-tag] [file-path:line] - [description]
2. [category-tag] [priority-tag] [file-path:line] - [description]
...

Category tags: [security], [bug], [async], [memory-leak], [resource-leak], [timeout], [shutdown], [edge-case], [type], [convention], [logging], [dependency], [rate-limit], [dead-code], [duplicate], [test], [practice], [docs], [chore]
Priority tags: [critical], [high], [medium], [low]
---
```

### Security Reviewer Prompt (name: "security-reviewer")

Append to the common preamble:

```
YOUR DOMAIN: Security & Authentication

Check for:
- OWASP A01: Broken Access Control — public endpoints intentional? Auth middleware on protected routes? IDOR prevention?
- OWASP A02: Secrets & Credentials — hardcoded secrets? Secrets in git? Sensitive data logged? Error messages leaking internals?
- OWASP A03: Injection — user input sanitized? Command injection? Path traversal? XSS in rendered content? SSRF (user-controlled URLs in server-side fetch)?
- OWASP A07: Authentication — tokens validated on every request? Auth middleware consistent? Session handling secure? Constant-time comparison for tokens/API keys (crypto.timingSafeEqual, not ===)?
- HTTPS & Transport — external calls use HTTPS? Certificate validation not disabled?
- Security Headers — CSP, X-Content-Type-Options, X-Frame-Options, HSTS configured?
- Rate limiting — API quotas handled? Backoff for 429s?
- Cookie security — httpOnly, secure, sameSite flags?

Search patterns (use Grep):
- `password|secret|api.?key|token` (case insensitive) — potential hardcoded secrets
- `eval\(|new Function\(` — dangerous code execution
- `exec\(|spawn\(` with variable input — command injection
- `fetch\(.*\$|fetch\(.*\+` — potential SSRF (user-controlled URLs)
- Log statements containing `password|secret|token|key|auth|headers|req\.body`

AI-Generated Code Risks:
- XSS vulnerabilities (2.74x higher in AI code)
- Missing input validation — AI often skips server-side validation
- Hallucinated security APIs — verify methods exist in the actual library
- Hallucinated packages — verify imports reference real packages in package.json
```

### Reliability Reviewer Prompt (name: "reliability-reviewer")

Append to the common preamble:

```
YOUR DOMAIN: Bugs, Async, Resources & Reliability

Check for:
- Logic errors — off-by-one, empty array/object edge cases, wrong comparisons, assignment vs comparison
- Null handling — nullable types without explicit handling, missing null checks
- Race conditions — shared state mutations, concurrent access without locks
- Async issues — promises without .catch(), async functions without try/catch, unhandled rejections, Promise.all error handling
- Memory leaks — unbounded arrays/Maps/Sets, event listeners without cleanup (.on without .off), timers without clearInterval, closures capturing large objects
- Resource leaks — connections not returned to pool, file handles not closed, streams not destroyed on error
- Timeout/hang scenarios — HTTP requests without timeout, API calls that could hang (Claude, Fitbit, Google), no circuit breaker
- Graceful shutdown — SIGTERM/SIGINT handlers? Cleanup on shutdown? Pending requests handled?
- Boundary conditions — empty inputs, single-element collections, max-size inputs, negative/zero values

Search patterns (use Grep):
- `\.then\(` without `.catch` nearby — unhandled promise
- `async ` functions — verify try/catch coverage
- `Promise\.all` — verify error handling
- `\.on\(` — event listeners (check for cleanup)
- `setInterval` — timers (check for clearInterval)
- `setTimeout` in loops — potential accumulation
- `new Map\(|new Set\(|\[\]` at module level — potential unbounded growth
```

### Quality Reviewer Prompt (name: "quality-reviewer")

Append to the common preamble:

```
YOUR DOMAIN: Type Safety, Conventions, Logging & Test Quality

Check for:
TYPE SAFETY:
- Unsafe `any` casts without justification
- Type assertions (`as Type`) that may be wrong
- Union types without exhaustive handling
- External data used without validation (API responses, AI outputs, file parsing)
- Missing runtime validation for API inputs

CLAUDE.md COMPLIANCE (read CLAUDE.md first!):
- Import path conventions (@/ alias)
- Naming conventions (files: kebab-case, components: PascalCase, etc.)
- Error response format compliance
- Server vs client component usage
- Any other project-specific rules

LOGGING:
- console.log/warn/error instead of proper logger
- Wrong log levels (errors at INFO, critical at DEBUG)
- Missing logs in error paths (empty catch blocks)
- Log overflow risks (logging in tight loops, large objects)
- Sensitive data in logs
- Missing structured logging fields

DEAD CODE & DUPLICATION:
- Unused functions, unreachable code
- Repeated logic that could be a single function
- Commented-out code blocks

TEST QUALITY (if tests exist):
- Tests with no meaningful assertions
- Tests that always pass
- Duplicate tests
- Mocks that hide real bugs
- No real customer data in tests

AI INTEGRATION (Claude API):
- Tool definitions have detailed descriptions (3-4+ sentences, the #1 performance lever)
- Tool parameters have descriptions with examples; constrained values use enums
- System prompts have clear role definition and tool usage guidance
- System prompts stay in sync with tool definitions (no stale references)
- stop_reason handled for all values (tool_use, end_turn, max_tokens)
- tool_result.tool_use_id matches tool_use.id; parallel results in ONE user message
- Agentic loops capped with max iterations (prevent infinite cycles)
- tool_use.input validated at runtime (don't trust AI output)
- max_tokens set appropriately (not too high, not truncation-prone)
- Token usage recorded (fire-and-forget, non-blocking)
- Claude API key from env var (not hardcoded, not logged)
- No user input injected raw into system prompts (prompt injection risk)
- AI-generated content sanitized before HTML rendering (XSS prevention)
- Tool results don't include raw secrets/tokens
- is_error: true set on tool_result for execution errors
- Rate limiting on Claude API endpoints

Search patterns (use Grep):
- `as any` — unsafe type cast
- `as unknown as` — double cast
- `@ts-ignore|@ts-expect-error` — suppressed type errors
- `console\.log|console\.warn|console\.error` — should use proper logger
- `catch\s*\([^)]*\)\s*\{[^}]*\}` — empty catch blocks
- `stop_reason` — verify all values handled
- `tool_use_id|tool_result` — verify ID matching
- `ANTHROPIC_API_KEY` — verify env-loaded, not logged
- `max_tokens` — verify reasonable limits
```

## Coordination (while reviewers work)

While waiting for reviewer messages:
1. Reviewer messages are **automatically delivered** to you — do NOT poll or manually check inbox
2. Teammates go idle after each turn — this is normal. An idle notification does NOT mean they are done. They are done when they send their findings message.
3. Track progress via `TaskList` — check which tasks are in progress vs completed
4. As each reviewer sends findings, acknowledge receipt
5. Wait until ALL 3 reviewers have reported before proceeding to merge

**If a reviewer gets stuck or stops without reporting:** Send them a message asking for their findings. If they don't respond, note that domain as "incomplete" in the final report.

## Merge & Deduplicate

Once all reviewer findings are collected:

### Validate existing issues

Combine validation results from all 3 reviewers:
- Issues reported as FIXED by any reviewer → close in Linear with comment
- Issues reported as STILL EXISTS → carry forward

### Classify pending existing issues

| Status | Criteria | Action |
|--------|----------|--------|
| `superseded` | New finding covers same issue | Close issue (new finding wins) |
| `needs_update` | Issue exists but line numbers or severity changed | Update issue description/priority |
| `still_valid` | Issue unchanged, no overlapping new finding | Keep as-is |

### Deduplicate new findings

- Same code location reported by multiple reviewers → merge into the one with higher priority
- Same root cause manifesting in multiple locations → create one issue covering all locations

### Reassess priorities

| | High Likelihood | Medium Likelihood | Low Likelihood |
|---|---|---|---|
| **High Impact** | Critical | Critical | High |
| **Medium Impact** | High | Medium | Medium |
| **Low Impact** | Medium | Low | Low |

## Create Linear Issues

For each new finding, use `mcp__linear__create_issue`:

```
team: [discovered team name]
state: "Backlog"
title: "[Brief description of the issue]"
description: (see Issue Description Format below)
priority: [1|2|3|4] (mapped from critical/high/medium/low)
labels: [Mapped label(s)]
```

**Issue Description Format:**

```
**Problem:**
[Clear, specific problem statement — 1-2 sentences]

**Context:**
[Affected file paths with line numbers, e.g. `src/lib/fitbit.ts:120-135`]

**Impact:**
[Why this matters — user-facing impact, data integrity, security risk, etc.]

**Acceptance Criteria:**
- [ ] [Specific, verifiable criterion — e.g. "API returns error response when DB insert fails after Fitbit log"]
- [ ] [Another criterion]
```

**Label Mapping:**

| Category Tags | Linear Label |
|---------------|--------------|
| `[security]`, `[dependency]` | Security |
| `[bug]`, `[async]`, `[shutdown]`, `[edge-case]`, `[type]`, `[logging]` | Bug |
| `[memory-leak]`, `[resource-leak]`, `[timeout]`, `[rate-limit]` | Performance |
| `[convention]` | Convention |
| `[dead-code]`, `[duplicate]`, `[test]`, `[practice]`, `[docs]`, `[chore]` | Technical Debt |

**Priority Mapping:**
- `[critical]` → 1 (Urgent)
- `[high]` → 2 (High)
- `[medium]` → 3 (Medium)
- `[low]` → 4 (Low)

**Rules:**
- NO solutions in issue descriptions — acceptance criteria define "done", not how to get there
- Include file paths with line numbers in Context
- One issue per distinct finding

## Shutdown Team

After all Linear issues are created:
1. Send shutdown requests to all 3 reviewers using `SendMessage` with `type: "shutdown_request"`
2. Wait for shutdown confirmations
3. Use `TeamDelete` to remove team resources

## Fallback: Single-Agent Mode

If `TeamCreate` fails (agent teams unavailable), perform the audit sequentially as a single agent:

1. **Inform user:** "Agent teams unavailable. Running audit in single-agent mode."
2. **Validate existing issues** — For each `pending_validation` issue, check if the referenced code still has the problem. Close fixed issues, carry forward valid ones.
3. **Systematic exploration** — Use Task tool with `subagent_type=Explore` to examine each discovered area. Look for:
   - Logic errors, null handling, race conditions
   - Security vulnerabilities (injection, missing auth, exposed secrets)
   - Unhandled edge cases and boundary conditions
   - Type safety issues (unsafe casts, unvalidated external data)
   - Dead or duplicate code
   - Memory leaks, resource leaks, async issues
   - Timeout/hang scenarios, graceful shutdown issues
   - Logging issues
   See [references/compliance-checklist.md](references/compliance-checklist.md) for detailed checks.
4. **CLAUDE.md compliance** — Check project-specific rules
5. **Merge, deduplicate, reprioritize** — Same process as team mode (see Merge & Deduplicate section)
6. **Create Linear issues** — Same process as team mode (see Create Linear Issues section)

## Error Handling

| Situation | Action |
|-----------|--------|
| Linear MCP not connected | STOP — tell user to run `/mcp` |
| No tsconfig.json or package.json | Use conventions: `src/`, `lib/`, `app/` |
| npm audit fails | Note skip, continue |
| CLAUDE.md doesn't exist | Skip project-specific checks (tell quality-reviewer) |
| Linear Backlog query fails | Continue with fresh audit (no existing issues) |
| No existing Backlog issues | Start fresh (skip validation in reviewer prompts) |
| TeamCreate fails | Switch to single-agent fallback mode |
| Reviewer stops without reporting | Send follow-up message, note domain as incomplete |
| Referenced file no longer exists | Mark issue as `fixed`, close in Linear |
| Cannot determine if issue is fixed | Keep as `still_valid` |
| Large codebase (>1000 files) | Tell reviewers to focus on `$ARGUMENTS` area or entry points |

## Rules

- **Analysis only** — Do NOT modify source code
- **No solutions** — Document problems, not fixes
- **Lead handles all Linear writes** — Reviewers NEVER create issues directly
- **Deduplicate before creating** — No duplicate issues in Linear
- **Be thorough** — Every file in scope must be checked

## Termination

Output this report and STOP:

```
## Code Audit Report

**Team:** 3 reviewers (security, reliability, quality)
[OR: **Mode:** single-agent (team unavailable)]
**Preserved:** P non-audit issues (features, improvements)

### Existing Backlog Issues

- A kept (still valid)
- B closed (fixed or superseded)
- C updated (description/priority changed)

### New Issues (ordered by priority)

| # | ID | Priority | Label | Title |
|---|-----|----------|-------|-------|
| 1 | PROJ-N1 | Urgent | Security | Brief title |
| 2 | PROJ-N2 | High | Bug | Brief title |
| ... | ... | ... | ... | ... |

X issues total | Duplicates merged: M | Findings dropped: N

Next step: Review Backlog in Linear and use `plan-backlog` to create implementation plans.
```

Do not ask follow-up questions. Do not offer to fix issues.
